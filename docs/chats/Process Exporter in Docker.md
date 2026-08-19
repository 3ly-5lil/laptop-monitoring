# Process Exporter in Docker — Problem, Solution, and Final Configuration

## 1. Problem

The goal was to run **process-exporter inside a Docker container** while collecting metrics for **system/host processes**, not only processes inside the container.

The main concern was whether mounting the host `/proc` was enough to let process-exporter see and measure host processes.

The setup also produced this kernel/AppArmor audit log:

```text
2026-08-19 18:04:31.443 UNK
priority=5
syslog_identifier=kernel
audit: type=1400 audit(1787151871.442:481): apparmor="DENIED" operation="ptrace" class="ptrace" profile="docker-default" pid=4304 comm="process-exporte" requested_mask="read" denied_mask="read" peer="unconfined"
```

This showed that Docker's default AppArmor profile was blocking process-exporter from using ptrace with read access.

---

## 2. Why the Problem Happened

A process-exporter running in a normal container sees the container's own PID namespace by default.

Therefore, without additional configuration:

Host

├── Host processes

│   ├── java

│   ├── firefox

│   ├── python

│   └── ...

│

└── process-exporter container

    └── sees container processes

Mounting `/proc` alone is not the complete solution.

The exporter needs access to the **host `/proc` filesystem**, and using the host PID namespace makes the process view consistent with the host.

Additionally, AppArmor can restrict operations needed to inspect processes even when `/proc` is mounted correctly.

The audit entry:

apparmor="DENIED"

operation="ptrace"

requested_mask="read"

means that the `docker-default` AppArmor profile was preventing process-exporter from reading another process through `ptrace`.

---

## 3. Host Process Metrics

Once process-exporter can correctly inspect the host, it can expose process-level metrics such as:

- CPU time
- CPU usage (calculated in PromQL)
- RSS/physical memory
- Virtual memory
- Thread counts
- Number of matching processes

Important metrics include:

namedprocess_namegroup_cpu_seconds_total

namedprocess_namegroup_memory_bytes

namedprocess_namegroup_threads

CPU usage can be calculated with:

100 * rate(namedprocess_namegroup_cpu_seconds_total[5m])

Approximately:

100% = one CPU core

200% = two CPU cores

600% = six CPU cores

Memory can be queried with:

namedprocess_namegroup_memory_bytes

and displayed in Grafana as MiB/GiB.

This makes process-exporter appropriate for a process-level dashboard, while cAdvisor remains useful for container-level metrics.

---

## 4. Recommended Docker Configuration

The basic process-exporter configuration should expose the host `/proc` and tell process-exporter to use it:

process-exporter:

  image: ncabatoff/process-exporter:latest

  container_name: process-exporter

  

  pid: host

  

  volumes:

    - /proc:/host/proc:ro

    - ./process-exporter.yml:/config/process-exporter.yml:ro

  

  command:

    - --procfs=/host/proc

    - --config.path=/config/process-exporter.yml

  

  ports:

    - "9256:9256"

The important pieces are:

pid: host

and:

- /proc:/host/proc:ro

with:

--procfs=/host/proc

This gives process-exporter access to the host's `/proc` rather than only the container's process information.

---

## 5. AppArmor Issue

The audit log confirms that Docker's default AppArmor profile was also restricting process inspection:

profile="docker-default"

operation="ptrace"

requested_mask="read"

denied_mask="read"

This means that having `/proc` mounted does **not automatically guarantee complete process visibility**.

A useful diagnostic configuration is:

process-exporter:

  image: ncabatoff/process-exporter:latest

  container_name: process-exporter

  

  pid: host

  

  security_opt:

    - apparmor=unconfined

  

  volumes:

    - /proc:/host/proc:ro

    - ./process-exporter.yml:/config/process-exporter.yml:ro

  

  command:

    - --procfs=/host/proc

    - --config.path=/config/process-exporter.yml

  

  ports:

    - "9256:9256"

Then recreate the container:

docker compose up -d --force-recreate process-exporter

And check whether AppArmor denials remain:

journalctl -k | grep 'process-exporte.*DENIED'

If the denials disappear and process metrics become complete, AppArmor was confirmed as the blocker.

### Security note

`apparmor=unconfined` should preferably be treated as a **diagnostic/test solution**, not automatically as the final production security configuration.

A more secure final setup would use a custom AppArmor profile that grants only the permissions process-exporter needs.

Do not disable AppArmor globally just to fix process-exporter.

---

## 6. Verification

Because the process-exporter image is minimal, commands such as:

docker exec -it process-exporter cat /host/proc/1/cmdline

may fail with:

OCI runtime exec failed: exec failed: unable to start container process:

exec: "cat": executable file not found in $PATH

This does **not** mean `/host/proc` is inaccessible.

It means the process-exporter image does not contain `cat`.

Instead, verify the Docker configuration from the host:

docker inspect process-exporter --format '{{json .Mounts}}' | jq

The mount should show:

/proc -> /host/proc

Check the PID mode:

docker inspect process-exporter --format '{{.HostConfig.PidMode}}'

Expected:

host

Check the exporter directly:

curl http://localhost:9256/metrics

or:

curl -s http://localhost:9256/metrics | grep namedprocess

A temporary debugging container can also be used to inspect the host `/proc` without modifying the exporter image:

docker run --rm -it \

  --pid=host \

  -v /proc:/host/proc:ro \

  alpine:latest \

  sh

Then:

ls /host/proc

cat /host/proc/1/cmdline

---

## 7. Final Monitoring Architecture

The intended monitoring architecture is:

                         HOST

                           │

          ┌────────────────┼────────────────┐

          │                │                │

          ▼                ▼                ▼

   node-exporter     process-exporter     cAdvisor

          │                │                │

          │                │                │

    Host metrics      Process metrics   Container metrics

          │                │                │

          └────────────────┼────────────────┘

                           ▼

                       Prometheus

                           │

                           ▼

                         Grafana

Responsibilities:

### node-exporter

Host-level metrics:

- CPU
- RAM
- Swap
- Disk
- Filesystem
- Network
- Sensors

### process-exporter

Process-level metrics:

- CPU
- Memory
- Threads
- Process counts
- Process groups

### cAdvisor

Container-level metrics:

- Container CPU
- Container memory
- Container network
- Container filesystem

This separation allows the Grafana dashboards to answer both:

> Which container is consuming resources?

and:

> Which actual process/application is consuming resources?

---

## 8. Final Conclusion

The original issue was not simply that process-exporter could not read `/proc`.

There were two separate requirements:

1. **Expose the host process information**
    - Use the host PID namespace.
    - Mount the host `/proc`.
    - Configure process-exporter with `--procfs=/host/proc`.
2. **Allow the exporter to inspect processes**
    - The Docker `docker-default` AppArmor profile was generating `ptrace` read denials.
    - This can prevent complete process inspection.
    - `apparmor=unconfined` can be used to confirm the diagnosis.
    - For a hardened final deployment, a dedicated AppArmor profile is preferable.

The final target is therefore:

pid: host

  

volumes:

  - /proc:/host/proc:ro

  

command:

  - --procfs=/host/proc

with the AppArmor restriction addressed separately.

The key audit message:

apparmor="DENIED" operation="ptrace" profile="docker-default"

was the important clue that `/proc` visibility and process inspection permissions were two different problems.