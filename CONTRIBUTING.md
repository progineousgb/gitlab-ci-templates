# Contributing

- Deploy templates define hidden jobs (`.pg-*`) so including a file never adds a job by itself.
- Configure through variables with defaults in the template, and document every variable in the README.
- Keep host key checking on. Never print secrets.
- Run `yamllint -s .` before opening a merge request, and test the change on a real pipeline.
