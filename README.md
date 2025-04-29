

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
  bare_repo_path="$IMPORT_DIR/${bundle_name}.git"
  working_repo_path="$IMPORT_DIR/${bundle_name}"

  echo "[INFO] Processing bundle: $bundle_name"

  # 1. Initialize bare repository
  mkdir -p "$bare_repo_path"
  git init --bare "$bare_repo_path"

  # 2. Fetch into bare repo
  if git --git-dir="$bare_repo_path" fetch "$bundle_path" 'refs/heads/*:refs/heads/*'; then
    echo "[INFO] Successfully fetched into bare repo: $bare_repo_path"

    # 3. Clone the bare repo into working directory
    git clone "$bare_repo_path" "$working_repo_path"

    echo "[INFO] Successfully cloned to working repo: $working_repo_path"
  else
    echo "[ERROR] Failed to fetch from bundle: $bundle_name. Skipping."
    rm -rf "$bare_repo_path"
  fi
done

echo "[INFO] GitLab Airgap Bundle Import Process Completed."

