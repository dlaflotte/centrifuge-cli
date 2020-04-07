# Centrifuge Policy Check

The `policy` subcommand is a tool that enables users to apply their own policy rules against a Centrifuge report.

This command line tool takes a policy file that defines the policy rules that you want to apply. It gathers report data via the Centrifuge REST API, for which you have to supply your Centrifuge API authentication token, and generates an output file containing the results of the policy as it applies to the given Centrifuge report.

## Prerequisites

Before you begin, ensure you have met the following requirements:
* You have python 3 installed (tested with python 3.7)
* You have an active Centrifuge account with a valid API authtoken
* You have a completed Centrifuge report that you wish to check
* You have installed the latest version of the Centrifuge command line tool (version 0.2.2 or later)

## Installing the Centrifuge CLI

Tested on Linux with python 3.7
```
pip3 install centrifuge-cli
```

## Using centrifuge-policy-check.py

```
# Check your policy against Centrifuge report 1234 and output CSV format (default)
centrifuge report --ufid=<REPORT ID> check-policy --policy-yaml my-policy.yml

# Check your policy against Centrifuge report 1234 and output json format
centrifuge --outfmt json report --ufid=<REPORT ID> check-policy --policy-yaml my-policy.yml
```


## Policy Rule Definition

A policy rules file must be specified in order for this tool to function.

The rules file follows a specific YAML format and allows the user to fully customize which rules to apply.  Only rules that are specified in the file will show passing or failing results. Rules that are not specified will show a result of `No policy specified`

The top level of the rules YAML specification contains the policy version and the rules object:
```
policyVersion: 1.0
rules: {}
```

Current policy schema version: `1.0`

Refer to example rule definition files included in this repository for more examples.

#### Rule exceptions

You may only want to apply a rule to a certain set of files rather than all the files in the firmware image, or apply the rule to all files except a certain set of files you wish to ignore. Many (if not all) of the rules can be defined in this way for maximum flexibility.

For example, this rule specification disallows all private keys:
```
  privateKeys:
    allowed: false
```
But you can also define it to disallow all private keys except those in the directory `/etc/ssl`:
```
  privateKeys:
    allowed: false
    exceptions:
      - /etc/ssl/*
```
Or you can define the rule to allow all private keys except those in the `/root` or `/opt/vendor` directories:
```
  privateKeys:
    allowed: true
    exceptions:
      - /root/*
      - /opt/vendor/*
 ```
 
## Rules

The following is a list of rules and their corresponding definition syntax.

### Rule: Expired Certificates

Firmware images with reported SSL certificates that are expired will fail this policy check.
```
  certificates:
    expired:
      allowed: false
      # optional list of files exempt from this rule (i.e. expired certificates in /etc/ssl are ok). 
      exceptions:
        - /etc/ssl/*
```

### Rule: Private Keys

Firmware images containing any private keys will fail this policy check.
```
  privateKeys:
    allowed: false
    # optional list of files that are allowed to have private keys
    exceptions:
      - /etc/ssl/*
```

### Rule: Weak Password Hashes

Firmware images containing any password hashes with the algorithms defined below will fail.
```
  passwordHashes:
    weakAlgorithms:
      - des
      - md5
```

### Rule: Code Flaws

Firmware images containing any high risk executables with code flaws will fail this policy check.
Set `allowed: false` to check against all files (except those omitted in the `exceptions` modifier.
Set `allowed: true` along with `exceptions` in order to only check a specific list of files for code flaws.
```
  code:
    flaws:
      allowed: true
    # the policy check will fail if any of the following files contain code flaws
    exceptions:
      - /usr/local/bin/*
      - /opt/vendor/*
```

### Rule: CVE Threshold

Firmware images containing any CVEs with a CVSS rating at or above the given threshold will fail the policy check.
```
  guardian:
    cvssScoreThreshold: 9.0
```

### Rule: Binary Hardness

Firmware images containing ELF binaries that lack the specified hardening features will fail the policy check.
In order to limit this check to a certain list of files, use the `include` modifier
```
  binaryHardening:
    requiredFeatures:
      - NX
      - PIE
      - RELRO
      - CANARY
      - STRIPPED
    # optional whitelist of binaries to check for above features. if omitted, all binaries will be checked
    include:
      - /opt/vendor/*
      - /root/*
      