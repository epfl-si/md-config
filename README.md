## PoC: Reusable Workflows
#### 🎯 Goal: The objective is to create reusable workflows for different CI processes.

**GitHub Documentation**: https://docs.github.com/en/actions/using-workflows/reusing-workflows#creating-a-reusable-workflow

Principle:

* A `called workflow` is defined in this repository. It utilizes the inputs and secrets keywords to define inputs or secrets that will be passed from a `caller workflow`.
* The `caller workflow` is defined in a separate repository.
  ```
  uses: epfl-si/md-infra/.github/workflows/frontend_ci.yml@main
  ```

For a detailed example, see https://github.com/epfl-si/edu-id-config/tree/main/.github/workflows