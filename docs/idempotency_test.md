# Idempotency Test

## Purpose

This test validates that the hardening role can be safely reapplied without making unnecessary changes after the target system already matches the desired configuration.

## Test Scope

* Host group: test
* Host: client01.example.test
* Playbook: playbooks/harden.yml
* Role: hardening

## Test Method

The hardening role was applied to the test group after the initial configuration run. A second run was then executed without making manual changes to confirm that the role did not continue modifying an already configured system.

## Result

* First run: completed successfully
* Second run: changed=0
* Result: PASS

## Validation Summary

The second run completed with `changed=0`, confirming that the role is idempotent against the test group. This indicates the role can enforce firewalld services, SELinux settings, fail2ban SSH protection, and fail2ban service state without repeatedly modifying a system that is already in the desired state.
