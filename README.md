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

#!/usr/bin/env bash

set -euo pipefail

echo "[INFO] Starting GitLab Airgap Bundle Import Process"

BUNDLE_DIR="${BUNDLE_DIR:-$(pwd)/gitlab_repos_bundles}"
IMPORT_DIR="${IMPORT_DIR:-$(pwd)/imported_gitlab_repos}"

mkdir -p "$IMPORT_DIR"

if ! ls "$BUNDLE_DIR"/*.bundle 1>/dev/null 2>&1; then
  echo "[WARNING] No bundle files found in $BUNDLE_DIR"
  exit 1
fi

for bundle_path in "$BUNDLE_DIR"/*.bundle; do
  bundle_name="$(basename "$bundle_path" .bundle)"
  sanitized_repo_path="$IMPORT_DIR/$bundle_name"

  echo "[INFO] Processing bundle: $bundle_name"

  # Clean repo setup
  mkdir -p "$sanitized_repo_path"
  pushd "$sanitized_repo_path" >/dev/null

  # Initialize git without creating a default branch
  git init --initial-branch=import-temp

  # Fetch from bundle
  if git fetch "$bundle_path" 'refs/heads/*:refs/heads/*'; then
    echo "[INFO] Successfully fetched from bundle: $bundle_name"

    # Remove the temporary branch if it exists
    if git show-ref --verify --quiet refs/heads/import-temp; then
      git branch -D import-temp
    fi

    # Find available branches
    mapfile -t branches < <(git for-each-ref --format='%(refname:short)' refs/heads/)

    if [[ ${#branches[@]} -eq 0 ]]; then
      echo "[ERROR] No branches found in bundle: $bundle_name. Skipping."
      popd >/dev/null
      continue
    fi

    # Checkout the first real branch
    git checkout "${branches[0]}"
    echo "[INFO] Checked out branch: ${branches[0]}"
  else
    echo "[ERROR] Failed to fetch from bundle: $bundle_name. Skipping."
  fi

  popd >/dev/null
done

echo "[INFO] GitLab Airgap Bundle Import Process Completed."
