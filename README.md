MODE=full ./gitlab_bundle_script.sh


#!/usr/bin/env bash
set -euxo pipefail

LOG_FILE="error.log"
: >"$LOG_FILE"
exec 2>>"$LOG_FILE"
exec > >(tee -a "$LOG_FILE")

# Validate GitLab token
if [[ -z "${TF_VAR_gitlab_token:-}" && -z "${GITLAB_TOKEN:-}" ]]; then
  echo "Error: GitLab token not set." >&2
  exit 1
fi

# MODE toggle: 'full' or 'incremental'
MODE="${MODE:-incremental}"

GITLAB_TOKEN="${TF_VAR_gitlab_token:-$GITLAB_TOKEN}"
GITLAB_URL="${GITLAB_URL:-https://gitlab.mda.mil}"
GROUP_ID="${GROUP_ID:-3482}"

CWD=$(pwd)
BUNDLE_DIR="${BUNDLE_DIR:-$CWD/gitlab_repos_bundles}"
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-$CWD/local_gitlab_repos}"
BUNDLE_BRANCHES="${BUNDLE_BRANCHES:-main staging}"
TAG_SUFFIX="transfer"

IFS=' ' read -r -a BUNDLE_BRANCHES_ARRAY <<< "$BUNDLE_BRANCHES"
mkdir -p "$BUNDLE_DIR" "$LOCAL_REPO_BASE"

get_projects_in_group() {
  curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$GITLAB_URL/api/v4/groups/$GROUP_ID/projects?include_subgroups=true&per_page=100" |
    jq -r '.[] | "\(.id);\(.path_with_namespace)"'
}

get_repo_https_url() {
  local repo_id=$1
  curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_URL/api/v4/projects/$repo_id" |
    jq -r '.http_url_to_repo'
}

get_latest_commit() {
  git rev-parse HEAD
}

get_root_commit() {
  git rev-list --max-parents=0 HEAD | tail -n 1
}

get_last_commit_file() {
  local sanitized=$1
  local branch=$2
  echo "$BUNDLE_DIR/last_commit__${sanitized}__${branch}"
}

create_bundle() {
  local repo_path=$1
  local branch=$2
  local bundle_file=$3
  local sanitized=$4

  echo "[MODE: $MODE] Processing $sanitized on branch $branch"

  cd "$repo_path"
  git checkout "$branch" || git checkout -b "$branch" "origin/$branch"
  git pull origin "$branch"

  local latest_commit
  latest_commit=$(get_latest_commit)

  local last_commit_file
  last_commit_file=$(get_last_commit_file "$sanitized" "$branch")

  local last_commit=""
  if [[ -f "$last_commit_file" ]]; then
    last_commit=$(cat "$last_commit_file")
    if ! git cat-file -e "$last_commit" 2>/dev/null; then
      last_commit=$(get_root_commit)
    fi
  else
    last_commit=$(get_root_commit)
  fi

  if [[ "$MODE" == "incremental" ]]; then
    if [[ "$latest_commit" == "$last_commit" ]]; then
      echo "[SKIPPED] No new commits for $sanitized on $branch"
      cd - >/dev/null
      return 0
    fi

    local tag="v$(date +%Y%m%d%H%M%S)-$TAG_SUFFIX"
    echo "[TAG] Creating transfer tag: $tag at $latest_commit"
    git tag -a "$tag" "$latest_commit" -m "Transfer tag $tag"
    git bundle create "$bundle_file" "$last_commit..$latest_commit"
    echo "$latest_commit" > "$last_commit_file"
    echo "[BUNDLE] Incremental bundle created at $bundle_file"
  else
    echo "[BUNDLE] Creating full bundle for $sanitized on $branch"
    git bundle create "$bundle_file" --all
    echo "$latest_commit" > "$last_commit_file"
  fi

  cd - >/dev/null
}

process_repo() {
  local repo_id=$1
  local repo_namespace=$2

  repo_url=$(get_repo_https_url "$repo_id")
  sanitized_name=$(echo "$repo_namespace" | sed 's|/|__|g')
  local_path="$LOCAL_REPO_BASE/$sanitized_name"

  if [[ -d "$local_path/.git" ]]; then
    git -C "$local_path" fetch --all --tags
  else
    git clone "$repo_url" "$local_path"
  fi

  for branch in "${BUNDLE_BRANCHES_ARRAY[@]}"; do
    if git -C "$local_path" show-ref --verify --quiet "refs/remotes/origin/$branch"; then
      git -C "$local_path" checkout "$branch" || git -C "$local_path" checkout -b "$branch" "origin/$branch"
      bundle_path="$BUNDLE_DIR/${sanitized_name}__${branch}.bundle"
      create_bundle "$local_path" "$branch" "$bundle_path" "$sanitized_name"
    else
      echo "[WARNING] Branch $branch not found in $repo_namespace"
    fi
  done
}

main() {
  REPOSITORIES=$(get_projects_in_group)
  if [[ -z "$REPOSITORIES" ]]; then
    echo "No repositories found."
    exit 1
  fi

  echo "$REPOSITORIES" | while IFS=';' read -r repo_id repo_namespace; do
    [[ -z "$repo_id" ]] && continue
    process_repo "$repo_id" "$repo_namespace"
  done

  echo "------- Summary of Bundles --------"
  if ls -1 "$BUNDLE_DIR"/*.bundle 2>/dev/null; then
    echo "Bundles created:"
    ls -l "$BUNDLE_DIR"/*.bundle
  else
    echo "No bundles were created."
  fi
}

main


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
./gitlab_airgap_bundle_import.sh

#!/usr/bin/env bash
set -euo pipefail


TARGET_REPO_NAMESPACE="https://gitlab.mda.mil/gm-tma/transfer-script-test-group/transfer-script-test-group.git"

LOG_FILE="error.log"
: > "$LOG_FILE"
exec 2>> "$LOG_FILE"
exec > >(tee -a "$LOG_FILE")

echo "[INFO] Starting GitLab Airgap Bundle Import Process"

if [ -z "${GITLAB_TOKEN:-}" ]; then
  echo "[ERROR] GITLAB_TOKEN environment variable is not set." >&2
  exit 1
fi


CWD=$(pwd)
BUNDLE_DIR="${BUNDLE_DIR:-$CWD/gitlab_repos_bundles}"
DEST_GITLAB_URL="${DEST_GITLAB_URL:-https://gitlab.mda.mil}"
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-$CWD/imported_gitlab_repos}"
TODAY=$(date +%Y%m%d)
BRANCH_PREFIX="transfer-${TODAY}/"

mkdir -p "$LOCAL_REPO_BASE"

# --- Helper functions ---

urlencode() {
  local string="$1"
  local strlen=${#string}
  local encoded=""
  for (( pos=0 ; pos<strlen ; pos++ )); do
    c=${string:$pos:1}
    case "$c" in
      [a-zA-Z0-9.~_-]) encoded+="$c" ;;
      *) encoded+=$(printf '%%%02X' "'$c") ;;
    esac
  done
  echo "$encoded"
}

create_branch_if_missing() {
  local project_id=$1
  local branch_name=$2
  local ref=$3

  local branch_exists
  branch_exists=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$DEST_GITLAB_URL/api/v4/projects/$project_id/repository/branches/$(urlencode "$branch_name")" | jq -r '.name')

  if [ "$branch_exists" == "null" ]; then
    echo "[INFO] Branch $branch_name does not exist. Creating from $ref"
    curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
      "$DEST_GITLAB_URL/api/v4/projects/$project_id/repository/branches" \
      --data-urlencode "branch=$branch_name" \
      --data-urlencode "ref=$ref" > /dev/null
  fi
}

submit_merge_request() {
  local project_id=$1
  local source_branch=$2
  local target_branch=$3
  local repo_name=$4

  echo "[INFO] Creating merge request from $source_branch to $target_branch"
  curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$DEST_GITLAB_URL/api/v4/projects/$project_id/merge_requests" \
    --data-urlencode "source_branch=$source_branch" \
    --data-urlencode "target_branch=$target_branch" \
    --data-urlencode "title=Merge from $source_branch into $target_branch" \
    --data-urlencode "description=Automated MR for repo $repo_name" \
    --data "remove_source_branch=true" > /dev/null
}

get_namespace_id() {
  local namespace_path=$1

  [ -z "$namespace_path" ] || [ "$namespace_path" == "." ] && echo "null" && return

  namespace=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$DEST_GITLAB_URL/api/v4/namespaces?search=$(urlencode "$namespace_path")" | jq -r '.[] | select(.full_path=="'"$namespace_path"'")')

  namespace_id=$(echo "$namespace" | jq -r '.id')
  if [ -z "$namespace_id" ] || [ "$namespace_id" == "null" ]; then
    echo "[INFO] Namespace $namespace_path does not exist. Creating..."
    namespace=$(curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
      "$DEST_GITLAB_URL/api/v4/groups" \
      --data-urlencode "name=$(basename "$namespace_path")" \
      --data-urlencode "path=$(basename "$namespace_path")" \
      --data-urlencode "parent_id=$(get_namespace_id "$(dirname "$namespace_path")")" \
      --data "visibility=private")

    namespace_id=$(echo "$namespace" | jq -r '.id')
  fi
  echo "$namespace_id"
}

push_to_transfer_branch() {
  local repo_namespace=$1
  local branch=$2

  echo "[INFO] Processing $repo_namespace:$branch"

  local sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's|/|__|g')
  local local_repo_path="$LOCAL_REPO_BASE/$sanitized_repo_namespace"

  cd "$local_repo_path" || { echo "[ERROR] Local repo $local_repo_path missing"; return; }

  dest_repo_url="$DEST_GITLAB_URL/$repo_namespace"

  project=$(curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$DEST_GITLAB_URL/api/v4/projects/$(urlencode "$repo_namespace")")

  project_id=$(echo "$project" | jq -r '.id')

  if [ "$project_id" == "null" ] || [ -z "$project_id" ]; then
    echo "[INFO] Project $repo_namespace not found. Creating..."
    project=$(curl --silent --request POST --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
      "$DEST_GITLAB_URL/api/v4/projects" \
      --data-urlencode "name=$(basename "$repo_namespace")" \
      --data-urlencode "path=$(basename "$repo_namespace")" \
      --data-urlencode "namespace_id=$(get_namespace_id "$(dirname "$repo_namespace")")" \
      --data "visibility=private")
    project_id=$(echo "$project" | jq -r '.id')
  fi

  # Always clean and reset remotes
  git remote remove origin 2>/dev/null || true
  git remote add origin "$dest_repo_url"

  transfer_branch="${BRANCH_PREFIX}${branch}"
  git push origin "$branch:$transfer_branch" || { echo "[ERROR] Push failed for $repo_namespace:$branch"; return; }
  create_branch_if_missing "$project_id" "$branch" "$transfer_branch"
  submit_merge_request "$project_id" "$transfer_branch" "$branch" "$repo_namespace"
  cd - >/dev/null
}

# --- MAIN LOOP ---

for bundle_file in "$BUNDLE_DIR"/*.bundle; do
  filename=$(basename "$bundle_file")
  repo_info="${filename%.bundle}"
  repo_namespace="$TARGET_REPO_NAMESPACE"
  branch=$(echo "$repo_info" | sed 's/^.*__//')

  sanitized_repo_namespace=$(echo "$repo_namespace" | sed 's|/|__|g')
  local_repo_path="$LOCAL_REPO_BASE/$sanitized_repo_namespace"

  mkdir -p "$local_repo_path"
  cd "$local_repo_path"

  git init

  echo "[INFO] Attempting to fetch bundle: $filename"
  if ! git fetch "$bundle_file"; then
    echo "[ERROR] Failed to fetch from bundle $filename. Skipping."
    cd - >/dev/null
    continue
  fi

  if ! git checkout -B "$branch" FETCH_HEAD 2>&1; then
    echo "[ERROR] Failed to checkout branch $branch from bundle $filename. Skipping."
    cd - >/dev/null
    continue
  fi

  push_to_transfer_branch "$repo_namespace" "$branch"

  cd - >/dev/null
done

echo "[COMPLETE] All repositories processed successfully." # added this agaian
