# Developer Notes – Forked strongswan

This [repository](https://github.com/atom-sdk/strongswan) is a fork of the official **[strongswan](https://github.com/strongswan/strongswan)** project.
The purpose of this fork is to introduce required enhancements while keeping the codebase as close as possible to upstream for easier maintenance and rebasing.

---

## Summary of Changes

The following updates have been applied on top of the upstream strongswan codebase:

### Hide tunnel logs
- Comment out android log print function `__android_log_print` from file `src/libcharon/plugins/android_log/android_log_logger.c` and `src/frontends/android/app/src/main/jni/libandroidbridge/charonservice.c`

### 16KB Memory Page Size Support
- Added support for **16KB memory page size**.

---

## Technical challenges
This project was successfully built on linux machine (ubuntu) as we were facing challenges on MacOS due to some shell commands. So, If you face challenges building after taking clone/pull, build on linux machines and maintain these changes mentioned earlier.

## Fork Maintenance Strategy

To ensure long-term maintainability and clean synchronization with upstream, the following workflow **must be followed**:

**Pull latest changes or Clone**
   ```bash
   git clone https://github.com/atom-sdk/strongswan
   git checkout master
   git checkout -b <branch-name>
   ```

When the required changes has been completed, push and create PR and merge into master. Also add new changes in **Summary of Changes** section.

### Upstream Compatibility Guidelines

- Do not reformat or refactor upstream code unless strictly necessary.
- Avoid changing public APIs without strong justification.
- Any upstream conflict resolution should prefer upstream behavior unless it breaks required functionality.
- Rebase regularly when upstream updates are pulled.
