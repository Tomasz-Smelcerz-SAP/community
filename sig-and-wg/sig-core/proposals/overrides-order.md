# Kyma Installer - user-defined overrides ordering

Created on 2018-09-10 by Tomasz Smelcerz(@Tomasz-Smelcerz-SAP) and Jakub Kabza (@jakkab).
This document is a part of the [Kyma modularization proposal](./modularization.md).

## Status

Proposed on 2018-09-10.

## Motivation

Installer allows to provide user-defined override values for Helm charts used in Kyma installation.
Users define these overrides in properly labelled ConfigMaps/Secrets deployed in `kyma-installer` namespace.
The overrides read from different objects are merged together into one structure, but the merging order is not defined.
Because of that it's hard to manage multiple override files used in different deployment scenarios.
Local installation on minikube (for development purposes), installation in CI (for testing), default cluster installation, custom cluster installation are examples of such scenarios.

## Goal

This proposal aims to enhance developer experience by:

- extending installation customizability
- reducing the number of entries in override files for different deployments
- avoiding redundant entries in override files for different deployments

## Current solution

Currently for handling just two installation scenarios: "local" (minikube) and "cluster",
users have two options:

- always install all override objects (both "local" and "cluster")
- only install override object specific for the installation scenario

First option means that users, even for local installation, must somehow provide overrides for cluster-specific options. Because these usually don't have sensible defaults in local installations, users commonly just use empty values (i.e. empty strings). It would be better to provide only the overrides that are necessary for specific installation.

Second option means there are two sets of files: ones for local installation, another for cluster installation. Users install "cluster" files only for cluster deployments.

We have two options here:
 - "local" files and "cluster" ones provide distinct set of overrides
 - "cluster" files contain "local" values as well as cluster-specific ones

If "local" files and "cluster" ones provide distinct set of overrides, cluster deployment requires to provide both sets.
But there's a problem here: If a "cluster" setup requires a change in some value already defined for "local" installation (e.g. azure-broker.enabled = true), it can't be done because Installer merges overrides in unknown order.
Lack of ordering means that "cluster" values could be overwritten by "local" ones, which is exactly the opposite to what user wants.

if "cluster" files contain "local" values as well as cluster-specific ones, users must only provide "cluster" files for cluster installation.
But there's a maintenance issue here: Values for "local" overrides are defined twice: once in "local" files, second time in "cluster" files, increasing maintenance effort.

These problems become worse with increasing number of deployment scenarios


## Suggested solution

- Users should be able to keep minimal set overides for given deployment (only the ones that are necessary)

- If a specific deployment is an extension of an existing one (e.g: "cluster" is an extension of "local"), users should be able to provide additional set of overrides that will "update" existing one in a predictable way.

In order to achieve that a concept of ordering between user-defined overrides is required.
To configure an order between user-defined override data objects, use the label: `install-order: <value>`, where `<value>` is a positive integer.
User-defined override objects are then applied in order: smaller `install-order` value are applied first.
The order between objects with the same `install-order` value is not determined.
Default `install-order` value (when the label is not set on an object) is zero.

#### Example:

Assuming three ConfigMaps:
- name: common, labels: [installer:overrides]
- name: cluster, labels: [installer:overrides, install-order: 10]
- name: extension, labels: [installer:overrides, install-order: 20]

Installer will build overrides staring with static overrides from "versions.yaml" file, if it exists (default behaviour).
Then values from "common" are applied to existing overrides (install-order = 0).
Afterwards "cluster" is applied to existing overrides (install-order = 10), then "extension" is applied to existing overrides (install-order = 20).

#### Merging details
  There are three cases to consider, when applying new set of overrides over an existing one:

- adding a new value
- changing existing value
- removing existing value

We support adding new values and changing existing ones, but we don't support removing of existing overrides.

