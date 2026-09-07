# Deploying the JPetStore QA sandbox

This fork runs as a practice target for the QA course at
`https://jpetstore.mate.academy`, on a shared sandbox EC2 box tagged
`conduit-jstore-jshop-sandboxes`, behind the utility ALB (target group
`mate-qa-jpetstore`, port 8080).

The box also hosts `conduit`, `coffeecart` (both under pm2) and `jshop`
(Docker). Only the units in this directory belong to JPetStore.

This repository is public, so the runbook below names resources by tag and
resolves ids at run time rather than publishing them:

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters 'Name=tag:Name,Values=conduit-jstore-jshop-sandboxes' \
            'Name=instance-state-name,Values=running' \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

The box is reached over **SSM only** — direct SSH is disallowed by org rule, and
no PEM key is needed:

```bash
aws ssm send-command --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["<cmd>"]' --query Command.CommandId --output text
aws ssm get-command-invocation --command-id <id> \
  --instance-id "$INSTANCE_ID" --query StandardOutputContent --output text
```

## Why systemd

JPetStore used to be started by hand and restarted weekly by an untracked
`~/projects/jpetstore-6/project_restart.sh` driven by an `ec2-user` crontab
entry. That arrangement failed in two ways, and on 2026-09-04 the site went
down for ~2.6 days as a result:

- **Nothing started it at boot.** The box rebooted on 2026-09-04 20:52 UTC.
  `conduit` and `coffeecart` came back via `pm2-ec2-user.service` and `jshop`
  via its Docker restart policy; JPetStore had no unit, no `rc.local` entry and
  no pm2 registration, so it stayed down and the ALB served 502s.
- **The weekly restart was a silent no-op.** The script called `./mvnw` without
  changing into its own directory, and cron runs with the home directory as its
  working directory, so `./mvnw` did not exist, `cargo:stop` failed and the
  `&&` short-circuited before `cargo:run`. The cron line redirected only
  stdout, so the error went to an unredirected stderr and
  `~/project_restart.log` sat at 0 bytes from 2025-04-14 onwards.

`WorkingDirectory` makes the first failure unexpressible, `WantedBy` covers the
reboot, `Restart=always` covers a JVM crash, and journald records what the
stdout-only redirect used to swallow.

`Environment=HOME` is the only variable needed: `mvnw` resolves Java from
`PATH`, and the box's default `java` is already Corretto 17, so no `JAVA_HOME`
is required.

`StartLimitInterval` / `StartLimitBurst` sit in `[Service]`, not `[Unit]`:
the box runs systemd 219 (Amazon Linux 2), and those directives only moved to
`[Unit]` in systemd 229. They matter because `RestartSec=15` exceeds the default
10-second start-limit window, so without them a permanently broken app would
restart every 15 seconds forever and never settle into `failed` — no state to
alert on, which is the same invisibility this change exists to remove.

## Before you touch anything: do not reset `pom.xml`

The box's `pom.xml` carries an **uncommitted** cargo `<deployables>` block that
this repository does not have:

```xml
<deployables>
  <deployable>
    ...
    <properties>
      <context>ROOT</context>
    </properties>
  </deployable>
</deployables>
```

`<context>ROOT</context>` is what deploys the app as `ROOT.war` and therefore
what makes `/actions/Catalog.action` — the students' URL *and* the ALB health
check path — resolve at all.

So a `git checkout -- pom.xml`, a fresh clone, or any pull that touches
`pom.xml` will leave `jpetstore.service` reporting `active` with no error in the
journal, while every student URL 404s and the ALB marks the target unhealthy.
Committing that block belongs in its own PR; until then, leave the box's
`pom.xml` alone.

The box's other local edits (`.gitignore`, `robots.txt`,
`src/main/webapp/index.html`, `IncludeTop.jsp`) are cosmetic.

## Install

### 1. Deliver the units

The box already has this repository checked out, so a fast-forward pull is the
whole copy step. None of the box's local edits overlap `deploy/`. `--ff-only`
is deliberate, so a future divergence fails loudly instead of opening a merge
on the box:

```bash
sudo -u ec2-user git -C /home/ec2-user/projects/jpetstore-6 pull --ff-only
cd /home/ec2-user/projects/jpetstore-6/deploy
```

### 2. Stop the hand-started instance first

**Do not skip this.** If JPetStore was started by hand (`mvnw cargo:run`), that
process is not in the new unit's cgroup, so `enable --now` starts a *second*
one. The new Tomcat then cannot bind 8080 and `Restart=always` retries every 15
seconds — and each attempt regenerates
`target/cargo/configurations/tomcat9x`, which is the **live** Tomcat's
`catalina.base`, `java.io.tmpdir` and logging config. That can take the site
down, which is the opposite of the point.

```bash
pgrep -u ec2-user -f 'MavenWrapperMain cargo:run'   # anything here is hand-started
sudo pkill -u ec2-user -f 'MavenWrapperMain cargo:run' || true
sudo pkill -u ec2-user -f 'catalina.base=.*tomcat9x' || true
sleep 10
ss -ltn | grep -q ':8080 ' && echo 'STILL BOUND - do not continue' || echo '8080 free'
```

Continue only once that prints `8080 free`.

### 3. Enable the units

```bash
sudo install -m 644 jpetstore.service /etc/systemd/system/jpetstore.service
sudo install -m 644 jpetstore-weekly-restart.service /etc/systemd/system/
sudo install -m 644 jpetstore-weekly-restart.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now jpetstore.service
sudo systemctl enable --now jpetstore-weekly-restart.timer
```

### 4. Retire the superseded crontab entry

The crontab is shared — it also carries `certbot renew` and conduit's
`db_maintenance.sh`. Back it up first: if `crontab -l` ever fails, the
right-hand side of the pipeline would install an **empty** crontab, and losing
certbot renewal there would stay invisible until the certificate expired.

```bash
sudo -u ec2-user crontab -l > /home/ec2-user/crontab.$(date +%F).bak
sudo -u ec2-user crontab -l | grep -v project_restart.sh | sudo -u ec2-user crontab -
sudo -u ec2-user crontab -l   # expect certbot + db_maintenance, and no project_restart
```

### 5. Retire `project_restart.sh`

Leaving it in place is a landmine rather than a neutral leftover: it is what
people on this box reach for when JPetStore misbehaves, and once systemd owns
the app, running it from the repository directory makes `cargo:stop` kill the
container out from under systemd while its `cargo:run` half races the unit for
port 8080. Replace its body with a pointer so muscle memory lands somewhere
useful:

```bash
sudo -u ec2-user tee /home/ec2-user/projects/jpetstore-6/project_restart.sh >/dev/null <<'EOF'
#!/bin/sh
echo "JPetStore is managed by systemd since 2026-09. Use:  sudo systemctl restart jpetstore" >&2
exit 1
EOF
```

The box's copy of that file is untracked, so this repository cannot enforce it —
it has to be done by hand during install.

## Verify

`Type=simple` reports `active` as soon as the Maven JVM is forked, well before
cargo has deployed the WAR and Tomcat has finished starting. Poll for readiness
rather than trusting `is-active` on its own:

```bash
systemctl is-enabled jpetstore.service          # enabled
systemctl list-timers jpetstore-weekly-restart  # next Monday 03:00 UTC
for i in $(seq 1 30); do
  curl -sf -o /dev/null http://localhost:8080/actions/Catalog.action && { echo up; break; }
  sleep 5
done
journalctl -u jpetstore -n 20 --no-pager        # expect "Tomcat 9.x started on port [8080]"
```

From anywhere, the public endpoint should return 200:

```bash
curl -sI https://jpetstore.mate.academy/actions/Catalog.action | head -1
```

ALB target health, resolved by target-group name (expect `healthy`):

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names mate-qa-jpetstore \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query 'TargetHealthDescriptions[].TargetHealth.State' --output text
```

## Notes

- `cargo:run` deploys the prebuilt `target/jpetstore.war`; it does not package.
  After changing application source, rebuild before restarting:
  `./mvnw package -DskipTests && sudo systemctl restart jpetstore`.
- The weekly restart reseeds the in-memory HSQLDB, which discards accounts
  students created during the week. That is the intended lifecycle for a
  practice target.
- The weekly unit uses `try-restart`, so it will not start the app if it has
  been stopped deliberately. The trade-off is that a deliberately-stopped app
  also skips its weekly reseed.
- `Persistent=false` is deliberate: a missed run must not fire on boot, where
  the app has just started fresh and a restart would be pointless.
- Terraform manages the instance (`services/utility-ec2` in the infrastructure
  repo) but sets `ignore_changes = [user_data]` and never sets `user_data`, and
  there is no configuration management, so these units are installed by hand
  via SSM.
