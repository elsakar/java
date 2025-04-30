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


++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Bundles created:
-rw-r--r-- 1 coder coder      3123 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__cloud-operations-deleted-2021__main.bundle
-rw-r--r-- 1 coder coder      4468 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-support__coder-blob-storage__main.bundle
-rw-r--r-- 1 coder coder      1006 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project__main.bundle
-rw-r--r-- 1 coder coder      1167 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project-security-policy-project__main.bundle
-rw-r--r-- 1 coder coder     23424 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__envbox__main.bundle
-rw-r--r-- 1 coder coder     23424 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__envbox__staging.bundle
-rw-r--r-- 1 coder coder     25283 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__general-vnc__main.bundle
-rw-r--r-- 1 coder coder     25283 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__general-vnc__staging.bundle
-rw-r--r-- 1 coder coder     22381 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__kasm-firmware-team__main.bundle
-rw-r--r-- 1 coder coder     22381 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__kasm-firmware-team__staging.bundle
-rw-r--r-- 1 coder coder     21976 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__kasm-general__main.bundle
-rw-r--r-- 1 coder coder     21976 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__kasm-general__staging.bundle
-rw-r--r-- 1 coder coder      2234 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__kubernetes-custom-security-policy-project__main.bundle
-rw-r--r-- 1 coder coder     20924 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__matlabclient__main.bundle
-rw-r--r-- 1 coder coder     20924 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__matlabclient__staging.bundle
-rw-r--r-- 1 coder coder     21926 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__ofrak__main.bundle
-rw-r--r-- 1 coder coder     21926 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__ofrak__staging.bundle
-rw-r--r-- 1 coder coder     21955 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__questa-vnc__main.bundle
-rw-r--r-- 1 coder coder     22725 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__questa-vnc-resize__main.bundle
-rw-r--r-- 1 coder coder     22725 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__questa-vnc-resize__staging.bundle
-rw-r--r-- 1 coder coder     21955 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__coder-templates__questa-vnc__staging.bundle
-rw-r--r-- 1 coder coder   2298826 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__containers__ana__pathfinders__mlflow_test__main.bundle
-rw-r--r-- 1 coder coder      1142 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__containers__code-marketplace__main.bundle
-rw-r--r-- 1 coder coder     28082 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__pipelines__azure-vm-pipeline__main.bundle
-rw-r--r-- 1 coder coder     12228 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__project-templates__coder-project-template__main.bundle
-rw-r--r-- 1 coder coder     12228 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__project-templates__coder-project-template__staging.bundle
-rw-r--r-- 1 coder coder     49337 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__renovate-runner__main.bundle
-rw-r--r-- 1 coder coder      5559 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__scripts__mcr-inspector__main.bundle
-rw-r--r-- 1 coder coder      8050 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__scripts__runner-check__main.bundle
-rw-r--r-- 1 coder coder  18731818 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__scripts__windows-scripts__main.bundle
-rw-r--r-- 1 coder coder      2557 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks_log_analytics__main.bundle
-rw-r--r-- 1 coder coder      8005 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks__main.bundle
-rw-r--r-- 1 coder coder      3239 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_core__main.bundle
-rw-r--r-- 1 coder coder      4425 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_session_hosts__main.bundle
-rw-r--r-- 1 coder coder      2626 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__key_vault__main.bundle
-rw-r--r-- 1 coder coder      2549 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__log_analytics__main.bundle
-rw-r--r-- 1 coder coder      2840 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__node_pool__main.bundle
-rw-r--r-- 1 coder coder      2611 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__postgres__main.bundle
-rw-r--r-- 1 coder coder      7709 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__storage_account__main.bundle
-rw-r--r-- 1 coder coder      2699 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__virtual_network__main.bundle
-rw-r--r-- 1 coder coder      2893 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__infrastructure__terraform__terraform-modules__azure-custom__vm__main.bundle
-rw-r--r-- 1 coder coder      3123 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__ana__requests__main.bundle
-rw-r--r-- 1 coder coder      3126 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__de__requests__main.bundle
-rw-r--r-- 1 coder coder      3126 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__ied__requests__main.bundle
-rw-r--r-- 1 coder coder      3127 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__mns__requests__main.bundle
-rw-r--r-- 1 coder coder      3125 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__requests__main.bundle
-rw-r--r-- 1 coder coder      5878 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__runner_sandbox__main.bundle
-rw-r--r-- 1 coder coder   2148781 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__svuml__main.bundle
-rw-r--r-- 1 coder coder     11309 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_binning__main.bundle
-rw-r--r-- 1 coder coder    462665 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_parse__main.bundle
-rw-r--r-- 1 coder coder   8655202 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__gws_dashking__main.bundle
-rw-r--r-- 1 coder coder     39247 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__ivv_gem_modem_uml__main.bundle
-rw-r--r-- 1 coder coder      3144 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb.dsc__main.bundle
-rw-r--r-- 1 coder coder     10736 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__cic__main.bundle
-rw-r--r-- 1 coder coder 136840764 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__gem_ublaze_microproc__main.bundle
-rw-r--r-- 1 coder coder     29889 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad5666__main.bundle
-rw-r--r-- 1 coder coder     18778 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7655__main.bundle
-rw-r--r-- 1 coder coder      5098 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7691__main.bundle
-rw-r--r-- 1 coder coder     29055 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7923__main.bundle
-rw-r--r-- 1 coder coder     54641 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7949__main.bundle
-rw-r--r-- 1 coder coder     49958 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7980__main.bundle
-rw-r--r-- 1 coder coder     16711 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9253__main.bundle
-rw-r--r-- 1 coder coder   1510971 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9786__main.bundle
-rw-r--r-- 1 coder coder    419257 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_adf411x__main.bundle
-rw-r--r-- 1 coder coder     14015 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ads8326__main.bundle
-rw-r--r-- 1 coder coder     10058 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv_eeprom__main.bundle
-rw-r--r-- 1 coder coder     44071 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_avalon__main.bundle
-rw-r--r-- 1 coder coder      9897 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_cad_l3h__main.bundle
-rw-r--r-- 1 coder coder    731795 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ds1626__main.bundle
-rw-r--r-- 1 coder coder   3431459 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_gige_temac_example__main.bundle
-rw-r--r-- 1 coder coder      4753 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max106__main.bundle
-rw-r--r-- 1 coder coder    199350 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max531__main.bundle
-rw-r--r-- 1 coder coder      5894 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pci__main.bundle
-rw-r--r-- 1 coder coder     20916 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pnor_flash__main.bundle
-rw-r--r-- 1 coder coder      6663 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_sdlc__main.bundle
-rw-r--r-- 1 coder coder   2803479 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_serpnor__main.bundle
-rw-r--r-- 1 coder coder     19927 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_spi__main.bundle
-rw-r--r-- 1 coder coder     10103 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__codec__main.bundle
-rw-r--r-- 1 coder coder     21852 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__sfpd__main.bundle
-rw-r--r-- 1 coder coder     23450 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio__main.bundle
-rw-r--r-- 1 coder coder     14503 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio_over_sfpdp__main.bundle
-rw-r--r-- 1 coder coder    357764 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_stft__main.bundle
-rw-r--r-- 1 coder coder      8312 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv2556__main.bundle
-rw-r--r-- 1 coder coder     13643 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv5638__main.bundle
-rw-r--r-- 1 coder coder     20571 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp422__main.bundle
-rw-r--r-- 1 coder coder   3612057 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp461__main.bundle
-rw-r--r-- 1 coder coder    514795 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tms320c67xx__main.bundle
-rw-r--r-- 1 coder coder     19020 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_uart__main.bundle
-rw-r--r-- 1 coder coder      7143 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublazefsl__main.bundle
-rw-r--r-- 1 coder coder      4758 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublaze_uvc__main.bundle
-rw-r--r-- 1 coder coder   1603649 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_wishbone__main.bundle
-rw-r--r-- 1 coder coder    696011 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qddrx__main.bundle
-rw-r--r-- 1 coder coder  13873515 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qpcie__main.bundle
-rw-r--r-- 1 coder coder   1257051 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc__main.bundle
-rw-r--r-- 1 coder coder 100197003 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__diamond__main.bundle
-rw-r--r-- 1 coder coder       466 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ise__main.bundle
-rw-r--r-- 1 coder coder    710785 Apr 30 17:17 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ivv_uvm-KycfX__main.bundle
-rw-r--r-- 1 coder coder  83064325 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__libero__main.bundle
-rw-r--r-- 1 coder coder 237705623 Apr 30 17:16 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__qformal_xilinx-SJyvL__main.bundle
-rw-r--r-- 1 coder coder 613292887 Apr 30 17:14 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main.bundle
-rw-r--r-- 1 coder coder 979557006 Apr 30 17:17 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main.bundle
-rw-r--r-- 1 coder coder    204950 Apr 30 17:17 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vsi_api-jDWEi__main.bundle
-rw-r--r-- 1 coder coder     28320 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main.bundle
-rw-r--r-- 1 coder coder      3764 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main.bundle
-rw-r--r-- 1 coder coder      3137 Apr 30 17:15 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__tenants__gmz__omd__tma__tma-issue-project__main.bundle
-rw-r--r-- 1 coder coder      3137 Apr 30 17:13 /home/coder/transfer-scripts/gitlab_repos_bundles/gm-tma__transfer-script-test-group__transfer-script-test-group__main.bundle
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

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

DEST_GITLAB_URL="${DEST_GITLAB_URL:-https://destination_gitlab_url}"
BUNDLE_DIR="${BUNDLE_DIR:-./gitlab_repos_bundles}"
LOCAL_REPO_BASE="${LOCAL_REPO_BASE:-./imported_gitlab_repos}"
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

  dest_repo_url="$DEST_GITLAB_URL/$repo_namespace.git"

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

  if ! git checkout -B "$branch" FETCH_HEAD; then
    echo "[ERROR] Failed to checkout branch $branch from bundle $filename. Skipping."
    cd - >/dev/null
    continue
  fi

  push_to_transfer_branch "$repo_namespace" "$branch"

  cd - >/dev/null
done

echo "[COMPLETE] All repositories processed successfully." # added this agaian

-------------------------------------------------------------------------------------------------------------------------------------------------------------------------

coder@rudy:~/transfer-scripts$ ./gitlab_airgap_bundle_import.sh 
[INFO] Starting GitLab Airgap Bundle Import Process
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__cloud-operations-deleted-2021__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__cloud-operations-deleted-2021__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-support__coder-blob-storage__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-support__coder-blob-storage__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__coder-templates-security-policy-project-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__envbox__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__envbox__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__envbox__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__envbox__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__general-vnc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__general-vnc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__general-vnc__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__general-vnc__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kasm-firmware-team__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kasm-firmware-team__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kasm-firmware-team__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kasm-firmware-team__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kasm-general__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kasm-general__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kasm-general__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kasm-general__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__kubernetes-custom-security-policy-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__kubernetes-custom-security-policy-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__matlabclient__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__matlabclient__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__matlabclient__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__matlabclient__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__ofrak__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__ofrak__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__ofrak__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__ofrak__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__questa-vnc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__questa-vnc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__questa-vnc-resize__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__questa-vnc-resize__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__questa-vnc-resize__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__questa-vnc-resize__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__coder-templates__questa-vnc__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__coder-templates__questa-vnc__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__containers__ana__pathfinders__mlflow_test__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__containers__ana__pathfinders__mlflow_test__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__containers__code-marketplace__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__containers__code-marketplace__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__pipelines__azure-vm-pipeline__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__pipelines__azure-vm-pipeline__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__project-templates__coder-project-template__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__project-templates__coder-project-template__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__project-templates__coder-project-template__staging.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__project-templates__coder-project-template__staging.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__renovate-runner__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__renovate-runner__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__mcr-inspector__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__mcr-inspector__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__runner-check__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__runner-check__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__scripts__windows-scripts__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__scripts__windows-scripts__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks_log_analytics__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks_log_analytics__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__aks__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_core__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_core__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_session_hosts__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__avd__avd_session_hosts__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__key_vault__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__key_vault__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__log_analytics__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__log_analytics__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__node_pool__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__node_pool__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__postgres__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__postgres__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__storage_account__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__storage_account__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__virtual_network__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__virtual_network__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__infrastructure__terraform__terraform-modules__azure-custom__vm__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__infrastructure__terraform__terraform-modules__azure-custom__vm__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__ana__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__ana__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__de__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__de__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__ied__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__ied__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__mns__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__mns__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__requests__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__requests__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__runner_sandbox__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__runner_sandbox__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__svuml__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__svuml__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_binning__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_binning__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_parse__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-automation__uvm_error_parse__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__gws_dashking__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__gws_dashking__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__ivv_gem_modem_uml__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__ivv_gem_modem_uml__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb.dsc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-gmn-gws__rxtx_tb.dsc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__cic__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__cic__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__gem_ublaze_microproc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__gem_ublaze_microproc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad5666__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad5666__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7655__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7655__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7691__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7691__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7923__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7923__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7949__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7949__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7980__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad7980__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9253__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9253__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9786__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ad9786__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_adf411x__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_adf411x__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ads8326__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ads8326__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv_eeprom__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_at28lv_eeprom__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_avalon__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_avalon__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_cad_l3h__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_cad_l3h__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ds1626__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ds1626__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_gige_temac_example__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_gige_temac_example__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max106__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max106__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max531__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_max531__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pci__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pci__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pnor_flash__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_pnor_flash__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_sdlc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_sdlc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_serpnor__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_serpnor__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_spi__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_spi__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__codec__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__codec__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__sfpd__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__sfpd__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio_over_sfpdp__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_srio_bundle__srio_over_sfpdp__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_stft__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_stft__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv2556__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv2556__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv5638__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tlv5638__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp422__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp422__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp461__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tmp461__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tms320c67xx__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_tms320c67xx__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_uart__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_uart__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublazefsl__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublazefsl__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublaze_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_ublaze_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_wishbone__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__ivv_wishbone__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qddrx__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qddrx__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qpcie__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__qpcie__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-uvcs__spacewire_uvc__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__diamond__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__diamond__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ise__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ise__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ivv_uvm-KycfX__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__ivv_uvm-KycfX__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__libero__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__libero__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__qformal_xilinx-SJyvL__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__qformal_xilinx-SJyvL__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__quartus__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vivado-l7h0x__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vsi_api-jDWEi__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__fw-vendors__vsi_api-jDWEi__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa_firmware__questa_job_mgr__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__swa__swa-software__ace-test__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__tenants__gmz__omd__tma__tma-issue-project__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__tenants__gmz__omd__tma__tma-issue-project__main.bundle. Skipping.
Reinitialized existing Git repository in /home/coder/transfer-scripts/imported_gitlab_repos/https:____gitlab.mda.mil__gm-tma__transfer-script-test-group__transfer-script-test-group.git/.git/
[INFO] Attempting to fetch bundle: gm-tma__transfer-script-test-group__transfer-script-test-group__main.bundle
[ERROR] Failed to fetch from bundle gm-tma__transfer-script-test-group__transfer-script-test-group__main.bundle. Skipping.
[COMPLETE] All repositories processed successfully.
