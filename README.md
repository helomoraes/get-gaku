# get-gaku
#!/usr/bin/env bash

: "${DESTINATION:=/usr/local/bin}"

readonly PROJECT="hackandsla.sh/gaku"
readonly CHECKSUM_FILE_NAME="checksums.txt"
readonly SUPPORTED_OS=("GNU/Linux")

set -o errexit   # abort on nonzero exitstatus
set -o nounset   # abort on unbound variable
set -o pipefail  # don't hide errors within pipes

main() {
  assert_destination_is_writable $DESTINATION
  assert_os_is_one_of "${SUPPORTED_OS[@]}"
  assert_commands_exists curl jq

  validate_gitlab_releases $PROJECT
  releases=$(get_gitlab_releases "${PROJECT}")
  latest_release=$(echo "${releases}" | jq ".|sort_by(.released_at)[-1]")

  project_name=$(get_project_name_from_project ${PROJECT})
  latest_version=$(echo "${latest_release}" | jq --raw-output  ".tag_name" )
  log_info "Downloading $(green "${project_name}") version $(green "${latest_version}")"

  log_info "Searching for architecture $(green "${HOSTTYPE}")"

  file_name="Linux_${HOSTTYPE}.tar.gz"
  download_link=$(echo "${latest_release}"\
    | jq --raw-output ".assets.links[] | select( .name | contains(\"${file_name}\")) | .url")
  if [[ -z "${download_link}" ]]; then
    log_error "Couldn't find download link for $(green "${HOSTTYPE}")"
    exit 1
  fi

  checksums=$(get_checksums "${latest_release}")
  checksum=$(echo "${checksums}" | grep "${file_name}" | cut -d ' ' -f 1)

  temp_dir=$(mktemp -d)
  trap 'rm --recursive --force "$temp_dir"' EXIT
  (
    cd "${temp_dir}"
    temp_file="${project_name}.tar.gz"
    curl  --silent --output "${temp_file}" "${download_link}"

    validate_checksum "${checksum}" "${temp_file}"

    tar --extract --file "${temp_file}"

    destination_file="${DESTINATION}/${project_name}"
    cp "${project_name}" "${destination_file}"

    log_info "Binary written to $(green "${destination_file}")"
  )
}

assert_destination_is_writable() {
  local destination="$1"

  if [[ ! -w "${destination}" ]]; then
    log_error "Error: current user doesn't have permissions to destination directory \
$(green "${destination}"). Please run with sudo or manually set a \
destination using the DESTINATION variable.\
\n\n\tExample:\
\n\tDESTINATION=/my/path $0"
    exit 1
  fi
}

assert_os_is_one_of() {
  local current_os
  current_os=$(uname --operating-system)
  local found_match=0

  for arg; do
    if [[ "${arg}" == "${current_os}" ]]; then
      found_match=1
    fi
  done

  if (( found_match == 0 )); then
      log_error "This script currently only supports the following platforms: $(green "$*")\
\n\tCurrent platform: $(green "${current_os}")"
      exit 1
  fi
}

get_project_name_from_project() {
  echo "${1##*/}"
}

validate_gitlab_releases() {
  local project="$1"

  local url
  url=$(get_gitlab_release_url "${project}")
  local http_code
  http_code=$(curl --silent --HEAD --output /dev/null -w "%{http_code}" "${url}" )
  if [[ "${http_code}" -ne 200 ]]; then
    local error_message
    error_message=$(get_gitlab_releases "${project}" | jq -r ".message")
    log_error "Error while downloading project $(green "${project}"): \
$(red "${error_message}")"
    exit 1
  fi
}

get_gitlab_releases() {
  local project="$1"

  local url
  url=$(get_gitlab_release_url "${project}")
  curl --silent "${url}"
}

get_gitlab_release_url() {
  local project="$1"

  local project_id
  project_id=$(urlencode "${project}")
  echo "https://gitlab.com/api/v4/projects/${project_id}/releases"
}

get_checksums() {
  local latest_release="$1"

  local checksum_link
  checksum_link=$(echo "${latest_release}" | jq --raw-output ".assets.links[] | select( .name==\"${CHECKSUM_FILE_NAME}\" ) | .url")
  if [[ -z "${checksum_link}" ]]; then
    log_error "Couldn't find checksum file $(green "${CHECKSUM_FILE_NAME}")"
    exit 1
  fi

  curl --silent "${checksum_link}"
}

validate_checksum() {
  local checksum="$1"
  local file="$2"

  local file_checksum
  file_checksum=$(sha256sum "${file}" | cut -d ' ' -f 1)

  if [[ "${file_checksum}" != "${checksum}" ]]; then
    log_error "File checksum doesn't match expected value. Try again later, or \
make an issue at https://gitlab.com/${PROJECT}/-/issues if the problem persists.\
\n\tGot:  $(green "${file_checksum}")\
\n\tWant: $(green "${checksum}")"
    exit 1
  fi
}

urlencode () {
    local i="$*"

    tab=$'\x9'
    i=${i//%/%25}  ; i=${i//' '/%20} ; i=${i//$tab/%09}
    i=${i//!/%21}  ; i=${i//'"'/%22}  ; i=${i//#/%23}
    i=${i//\$/%24} ; i=${i//\&/%26}  ; i=${i//\'/%27}
    i=${i//(/%28}  ; i=${i//)/%29}   ; i=${i//\*/%2a}
    i=${i//+/%2b}  ; i=${i//,/%2c}   ; i=${i//-/%2d}
    i=${i//\./%2e} ; i=${i//\//%2f}  ; i=${i//:/%3a}
    i=${i//;/%3b}  ; i=${i//</%3c}   ; i=${i//=/%3d}
    i=${i//>/%3e}  ; i=${i//\?/%3f}  ; i=${i//@/%40}
    i=${i//\[/%5b} ; i=${i//\\/%5c}  ; i=${i//\]/%5d}
    i=${i//\^/%5e} ; i=${i//_/%5f}   ; i=${i//\`/%60}
    i=${i//\{/%7b} ; i=${i//|/%7c}   ; i=${i//\}/%7d}
    i=${i//\~/%7e}
    echo "${i}"
}

assert_commands_exists() {
    local missing_commands=0
    for arg; do
        if [[ ! "$(command -v "${arg}")" ]]; then
            log_error "Command '${arg}' not found"
            missing_commands=1
        fi
    done

    if (( missing_commands == 1 )); then
        echo "Please install all required packages and try again" >&2
        exit 1
    fi
}

log_info() {
    echo -e "[$(green +)] $*"
}

log_error() {
    echo -e "[$(red -)] $*" >&2
}

green() {
  echo "$(tput setaf 2)$*$(tput sgr0)"
}

red() {
  echo "$(tput setaf 1)$*$(tput sgr0)"
}

main "$@"
