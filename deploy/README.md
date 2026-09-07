# Deploying the JPetStore QA sandbox

This fork runs as a practice target for the QA course at
`https://jpetstore.mate.academy`, on the shared sandbox EC2 box
`i-08d3e4dea44452167` (`conduit-jstore-jshop-sandboxes`), behind
`mate-utility-alb` (target group `mate-qa-jpetstore`, port 8080).

The box also hosts `conduit`, `coffeecart` (both under pm2) and `jshop`
(Docker). Only the units in this directory belong to JPetStore.

The box is reached over **SSM only** — direct SSH is disallowed by org rule, and
no PEM key is needed:

```bash
aws ssm send-command --instance-ids i-08d3e4dea44452167 \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["<cmd>"]' --query Command.CommandId --output text
aws ssm get-command-invocation --command-id <id> \
  --instance-id i-08d3e4dea44452167 --query StandardOutputContent --output text
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

## Install

Copy the three units to the box, then enable them:

```bash
sudo install -m 644 jpetstore.service /etc/systemd/system/jpetstore.service
sudo install -m 644 jpetstore-weekly-restart.service /etc/systemd/system/
sudo install -m 644 jpetstore-weekly-restart.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now jpetstore.service
sudo systemctl enable --now jpetstore-weekly-restart.timer
```

Then retire the superseded crontab entry, so the app is not restarted twice:

```bash
sudo -u ec2-user crontab -l | grep -v project_restart.sh | sudo -u ec2-user crontab -
```

`project_restart.sh` itself can stay on the box; nothing invokes it once the
crontab line is gone.

## Verify

```bash
systemctl is-enabled jpetstore.service          # enabled
systemctl is-active jpetstore.service           # active
systemctl list-timers jpetstore-weekly-restart  # next Monday 03:00 UTC
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/actions/Catalog.action
```

From anywhere, the public endpoint should return 200:

```bash
curl -sI https://jpetstore.mate.academy/actions/Catalog.action | head -1
```

ALB target health (expect `healthy`):

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:eu-central-1:781122033386:targetgroup/mate-qa-jpetstore/dee6056fc8f5eb70 \
  --query 'TargetHealthDescriptions[].TargetHealth.State' --output text
```

## Notes

- `cargo:run` deploys the prebuilt `target/jpetstore.war`; it does not package.
  After changing application source, rebuild before restarting:
  `./mvnw package -DskipTests && sudo systemctl restart jpetstore`.
- The weekly restart reseeds the in-memory HSQLDB, which discards accounts
  students created during the week. That is the intended lifecycle for a
  practice target.
- `Persistent=false` is deliberate: a missed run must not fire on boot, where
  the app has just started fresh and a restart would be pointless.
- The box's working copy carries uncommitted edits to `.gitignore`, `pom.xml`,
  `robots.txt`, `src/main/webapp/index.html` and
  `src/main/webapp/WEB-INF/jsp/common/IncludeTop.jsp`. They are not in this
  repo, so `git pull` on the box will not reproduce the running site. Committing
  them is separate work.
- Terraform manages the instance
  (`production/eu-central-1/services/utility-ec2/ec2.tf` in the infrastructure
  repo) but sets `ignore_changes = [user_data]` and there is no configuration
  management, so these units are installed by hand via SSM.
