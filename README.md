# packer-runner-template
Opinionated thin-runner template for Packer image-build repositories that overlay consumer inventory onto a SHA-pinned packer-framework. Data-only by design — runners own packer inventory and caller workflows; the framework owns executable build logic.
