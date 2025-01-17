# Chadburn: A Modern Job Scheduler

[![GitHub version](https://badge.fury.io/gh/PremoWeb%2FChadburn.svg)](https://github.com/PremoWeb/Chadburn/releases) ![Testing Status](https://github.com/PremoWeb/Chadburn/workflows/Testing%20Status/badge.svg)

**Chadburn** is a lightweight job scheduler designed for __Docker__ environments, developed in Go. It serves as a contemporary replacement for the traditional [cron](https://en.wikipedia.org/wiki/Cron).

---

### Special Note

Chadburn is a project built upon the ongoing development from Ofelia, a fork initiated by @rdelcorro. This project was created to address specific needs, including:

- Automatic task updates when Docker containers are started, stopped, restarted, or modified.
- Elimination of the need for a dummy task in the Chadburn container.
- Concurrent support for both INI files and Docker labels, allowing configurations to be merged seamlessly.
- The ability to recognize new tasks or remove existing ones without needing a restart.

---

### Why Choose Chadburn?

Since the release of [`cron`](https://en.wikipedia.org/wiki/Cron) by AT&T Bell Laboratories in March 1975, much has changed in the computing landscape, especially with the rise of Docker. While **Vixie’s cron** remains functional, it lacks extensibility and can be challenging to debug when issues arise.

Various solutions exist, including containerized cron implementations and command wrappers, but these often complicate straightforward tasks.

---

### Key Features

Chadburn's primary feature is its ability to execute commands directly within Docker containers. Utilizing Docker's API, Chadburn mimics the behavior of [`exec`](https://docs.docker.com/reference/commandline/exec/), enabling commands to run inside active containers. Additionally, it allows for command execution in new containers, which are destroyed after use.

---

## Configuration

A comprehensive wiki is underway to detail Chadburn's usage. Caprover users will soon have access to a One Click App for deploying and managing scheduled jobs via Service Label Overrides.

For others, here’s a quick guide to get started with Chadburn:

### Job Scheduling

Chadburn uses a scheduling format consistent with the Go implementation of `cron`. Examples include `@every 10s` or `0 0 1 * * *` (which runs every night at 1 AM).

**Note**: The scheduling format previously included seconds; however, this has been updated in the latest version of Chadburn. Significant development is planned to resolve various issues reported with both Ofelia and Chadburn.

You can configure four types of jobs:

- `job-exec`: Executes a command inside a running container.
- `job-run`: Runs a command in a new container using a specified image.
- `job-local`: Executes a command on the host running Chadburn.
- `job-service-run`: Runs a command inside a new "run-once" service for swarm environments.

For detailed parameters, refer to the [Jobs reference documentation](docs/jobs.md).

#### INI Configuration

To run Chadburn with an INI file, use the command:

```bash
chadburn daemon --config=/path/to/config.ini
```

Here’s a sample INI configuration:

```ini
[job-exec "job-executed-on-running-container"]
schedule = @hourly
container = my-container
command = touch /tmp/example

[job-run "job-executed-on-new-container"]
schedule = @hourly
image = ubuntu:latest
command = touch /tmp/example

[job-local "job-executed-on-current-host"]
schedule = @hourly
command = touch /tmp/example

[job-service-run "service-executed-on-new-container"]
schedule = 0,20,40 * * * *
image = ubuntu
network = swarm_network
command = touch /tmp/example
```

#### Docker Label Configurations

For Docker label configurations, Chadburn needs access to the Docker socket:

```bash
docker run -it --rm \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    premoweb/chadburn:latest daemon
```

The labels format is: `chadburn.<JOB_TYPE>.<JOB_NAME>.<JOB_PARAMETER>=<PARAMETER_VALUE>`. This configuration method supports all capabilities provided by INI files.

To execute `job-exec`, the target container must have the label `chadburn.enabled=true`.

For example, to run the `uname -a` command in an existing container called `my_nginx`, start `my_nginx` with the following configurations:

```bash
docker run -it --rm \
    --label chadburn.enabled=true \
    --label chadburn.job-exec.test-exec-job.schedule="@every 5s" \
    --label chadburn.job-exec.test-exec-job.command="uname -a" \
    nginx
```

Alternatively, you can use Docker Compose:

```yaml
version: "3"
services:
  chadburn:
    image: premoweb/chadburn:latest
    depends_on:
      - nginx
    command: daemon
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  nginx:
    image: nginx
    labels:
      chadburn.enabled: "true"
      chadburn.job-exec.datecron.schedule: "@every 5s"
      chadburn.job-exec.datecron.command: "uname -a"
```

#### Dynamic Docker Configuration

Chadburn can be run in its own container or directly on the host. It will automatically detect any containers that start, stop, or change, utilizing the labeled containers for dynamic task management.

#### Hybrid Configuration (INI + Docker)

You can combine INI files and Docker labels to manage configurations. Use INI files for global settings or tasks that cannot be defined solely through labels, while Docker labels can be employed for dynamically managed tasks.

```ini
[global]
slack-webhook = https://myhook.com/auth

[job-run "job-executed-on-new-container"]
schedule = @hourly
image = ubuntu:latest
command = touch /tmp/example
```

Use Docker labels for dynamic jobs:

```bash
docker run -it --rm \
    --label chadburn.enabled=true \
    --label chadburn.job-exec.test-exec-job.schedule="@every 5s" \
    --label chadburn.job-exec.test-exec-job.command="uname -a" \
    nginx
```

### Logging

Chadburn offers three logging drivers that can be configured in the `[global]` section:

- `mail` to send notifications via email.
- `save` to save structured execution reports in a specified directory.
- `slack` to send messages through a Slack webhook.

#### Logging Options

- `smtp-host`, `smtp-port`, `smtp-user`, `smtp-password`: SMTP server settings for email notifications.
- `email-to`, `email-from`: Sender and receiver email addresses.
- `mail-only-on-error`: Send email notifications only for failed executions.
- `insecure-skip-verify`: Skip SSL verification for SMTP (available in version 1.0.2+).

- `gotify-webhook`: URL for Gotify notifications.
- `gotify-only-on-error`: Send Gotify messages only for failures.
- `gotify-priority`: Priority level for Gotify messages.

- `save-folder`: Directory for storing execution reports.
- `save-only-on-error`: Save reports only for failed executions.

- `slack-webhook`: URL for Slack notifications.
- `slack-only-on-error`: Send Slack messages only for failures.

### Overlap Prevention

Chadburn prevents jobs from running concurrently if a previous execution has not yet completed. If a job has the `no-overlap` option enabled, it will not run multiple instances simultaneously.

---

## Installation

The simplest way to deploy **Chadburn** is using Docker, as outlined above.

If you prefer not to use the provided Docker image, you can download a binary from the [releases](https://github.com/PremoWeb/Chadburn/releases) page.

---

### Special Note for Caprover PaaS Users

Chadburn is available as a One Click App in the official Caprover app repository. After deployment, you can configure the scheduler for your apps using the Service Override section in your app’s configuration:

```yaml
TaskTemplate:
  ContainerSpec:
    Labels:
      chadburn.enabled: "true"
      chadburn.job-exec.rotate-puzzles.command: "php /var/www/rotate_games.php"
      chadburn.job-exec.rotate-puzzles.schedule: "@every 10m"
```

---

### Acknowledgments

We extend our gratitude to the Ofelia team and its contributors, particularly [@rdelcorro](https://github.com/rdelcorro) for addressing the issues highlighted in [this pull request](https://github.com/mcuadros/ofelia/pull/137).

Thank you to the original authors and contributors of Ofelia for their foundational work.

--- 

Feel free to modify or expand on any sections as needed!