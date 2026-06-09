# PayOS TCP Server Documentation

## Overview

This documentation set explains the TCP transport module of PayOS.

Use it when you need to:
- understand how the TCP server is discovered and started
- configure TCP listeners in `bootstrap.json`
- build custom decoder, encoder, or handler plugins
- understand how inbound TCP payloads become PayOS `Request` objects

## Documents

- [Architecture](./architecture.md)
- [Configuration](./configuration.md)
- [Request Lifecycle](./request-lifecycle.md)
- [Plugin Development](./plugin-development.md)

## Reading Order

Recommended order for a new developer:

1. Read `architecture.md` to understand the role of the module.
2. Read `request-lifecycle.md` to follow a TCP message end to end.
3. Read `configuration.md` if you need to run or deploy it.
4. Read `plugin-development.md` if you need a custom protocol adapter.
