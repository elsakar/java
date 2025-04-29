#!/usr/bin/env bash

set -euo pipefail

echo "[INFO] Starting GitLab Airgap Bundle Import Process"

BUNDLE_DIR="${BUNDLE_DIR:-$(pwd)/gitlab_repos_bundles}"
IMPORT_DIR="${IMPORT_DIR:-$(pwd)/imported_gitlab_repos}"

mkdir -p "$IMPORT_DIR"

for bundle_path in "$BUNDLE_DIR"/*.bundle; do
  [[ -e "$bundle_path" ]] || { echo "[WARNING] No bundle files found in $BUNDLE_DIR"; exit 1; }

  bundle_name=$(basename "$bundle_path" .bundle)
  sanitized_repo_path="$IMPORT_DIR/$bundle_name"

  echo "[INFO] Processing bundle: $bundle_name"

  # Create new repo directory
  mkdir -p "$sanitized_repo_path"
  cd "$sanitized_repo_path"

  git init

  if git fetch "$bundle_path" 'refs/heads/*:refs/heads/*'; then
    echo "[INFO] Successfully fetched from bundle: $bundle_name"

    # List available branches in the bundle
    branches=($(git for-each-ref --format='%(refname:short)' refs/heads/))

    if [[ ${#branches[@]} -eq 0 ]]; then
      echo "[ERROR] No branches found in bundle: $bundle_name. Skipping."
      cd - >/dev/null
      continue
    fi

    # Checkout the first branch (usually 'main' or 'staging')
    git checkout "${branches[0]}"
    echo "[INFO] Checked out branch: ${branches[0]}"
  else
    echo "[ERROR] Failed to fetch from bundle $bundle_name. Skipping."
  fi

  cd - >/dev/null
done

echo "[INFO] GitLab Airgap Bundle Import Process Completed."



++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main. Skipping.
[INFO] Processing bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main
Initialized empty Git repository in /home/coder/transfer-phase2/imported_gitlab_repos/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main/.git/
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main. Skipping.
[INFO] Processing bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main
Initialized empty Git repository in /home/coder/transfer-phase2/imported_gitlab_repos/gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main/.git/
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main. Skipping.
[INFO] Processing bundle: gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main
Initialized empty Git repository in /home/coder/transfer-phase2/imported_gitlab_repos/gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main/.git/
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main. Skipping.
[INFO] Processing bundle: gm-tma__tenants__gmz__omd__tma__tma-issue-project__main
Initialized empty Git repository in /home/coder/transfer-phase2/imported_gitlab_repos/gm-tma__tenants__gmz__omd__tma__tma-issue-project__main/.git/
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__tma__tma-issue-project__main. Skipping.
[INFO] Processing bundle: gm-tma__transfer-script-test-group__transfer-script-test-group__main
Initialized empty Git repository in /home/coder/transfer-phase2/imported_gitlab_repos/gm-tma__transfer-script-test-group__transfer-script-test-group__main/.git/
fatal: Refusing to fetch into current branch refs/heads/main of non-bare repository
[ERROR] Failed to fetch from bundle gm-tma__transfer-script-test-group__transfer-script-test-group__main. Skipping.
[INFO] GitLab Airgap Bundle Import Process Completed.
coder@rudy:~/transfer-phase2$ 
