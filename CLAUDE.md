# Titanium Web Proxy

## Project Overview
A lightweight, open-source HTTP(S) proxy server written in C#. This is a **forked/vendored** third-party library used as a project dependency by `ADBPhoneNetCore` for intercepting and managing HTTP traffic from Android devices.

## Tech Stack
- **.NET Standard 2.0+ / .NET Framework 4.5+**
- Multithreaded async proxy with connection pooling

## Features
- View, modify, redirect, and block HTTP requests/responses
- HTTPS decryption via MITM with auto-generated certificates
- SOCKS4/5 proxy support
- Mutual SSL authentication
- Kerberos/NTLM authentication support

## Build
```bash
cd src
dotnet build Titanium.Web.Proxy.sln
```

## Project Structure
```
/src/Titanium.Web.Proxy  — Main proxy library source
/examples                — Console and WPF example applications
/tests                   — Unit tests
/docs                    — API documentation
README.md                — Full usage documentation with code examples
```

## Usage in This Ecosystem
Referenced as a project dependency by `ADBPhoneNetCore` to provide local proxy server functionality for routing Android device traffic through the automation system.

## Key Conventions
- Upstream repo: [justcoding121/titanium-web-proxy](https://github.com/justcoding121/titanium-web-proxy)
- Do NOT modify core proxy logic unless necessary — prefer upstream patches
- Namespace: `Titanium.Web.Proxy`
