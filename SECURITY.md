# Security Policy

Use GitHub private vulnerability reporting for `Ezio2000/jharness-tools` when a
preset implementation, dependency, path boundary, network boundary, or result handling
issue could cause unsafe execution or data exposure. Do not publish credentials, user
data, or exploit details in a public issue.

Runtime contract vulnerabilities belong in `Ezio2000/jharness-python`; portable
specification ambiguities belong in `Ezio2000/jharness`.

Agent presets do not create a security boundary. The application-provided
`AgentBackend` must authorize every lookup, wait, and cancellation; derive Child
Runtime tools, approvals, credentials, workspace, network access, depth, and remaining
budgets from trusted Host policy; and avoid leaking whether an unauthorized Agent id
exists. Model-provided prompts and ids must never select unrestricted Runtime objects
or bypass inherited policy.
