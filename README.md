# Fuse Network FIP11 Validator Jailing

This is a brief commentary on how to design and implement the FIP11 of Fuse Network for validator Jailing problem. 
https://github.com/fuseio/FIPs/blob/master/FIPS/fip-11.md

## Table of Contents
### Fuse Network's Goals
### Identify benign VS malicious misbehaving Validator
### A new contract for Validator Statistics
### A ReportingContract for reporting benign and malicious behavior of validators


## Fuse Network's Goals:
Remove any misbehaving validators from the current and future validator set until they signal to the consensus that they are back online.

This will:

    - Ensure the block time is consistent.
    - Ensure we have maximum transaction throughput.
    - Ensure all transactions times are <5seconds.

