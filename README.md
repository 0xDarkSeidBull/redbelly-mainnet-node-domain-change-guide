#!/usr/bin/env bash
# switch-domain.sh: move a Redbelly mainnet node to a new domain without resyncing.
# Guide: https://github.com/0xDarkSeidBull/redbelly-mainnet-node-domain-change-guide
#
#   sudo bash switch-domain.sh check  NEW_DOMAIN          read-only report, changes nothing
#   sudo bash switch-domain.sh cert   NEW_DOMAIN [EMAIL]  get the SSL certificate (node keeps running)
#   sudo bash switch-domain.sh apply  NEW_DOMAIN          switch the node (only after Redbelly support confirms)
#   sudo bash switch-domain.sh status NEW_DOMAIN          service, certificate and sync status
#
# The script never touches the chain data and never prints private keys.

if [ -z "${BASH_VERSION:-}" ]; then
  echo "Run this script with bash: sudo bash switch-domain.sh ..."
  exit 1
fi
set -u
umask 077

VERSION=2026-11-24
SERVICE=redbelly.service
RB_ETC=/etc/redbelly
LE=/etc/letsencrypt
BACKUP_ROOT=/root
HOME_DIRS="/root /home"
DEFAULT_LOG=/var/log/redbelly/rbn_logs/rbbc_logs.log
RPC_URL=https://governors.mainnet.redbelly.network
TLS_PORT=1111
CERTS_ROOT=$RB_ETC/certs

FAILS=0
TODOS=0
ok()      { echo "  OK    $*"; }
warn()    { echo "  WARN  $*"; }
bad()     { echo "  FAIL  $*"; FAILS=$((FAILS + 1)); }
todo()    { echo "  TODO  $*"; TODOS=$((TODOS + 1)); }
note()    { echo "        $*"; }
section() { echo; echo "== $* =="; }
die()     { echo; echo "STOPPED: $*"; exit 1; }
have()    { command -v "$1" >/dev/null 2>&1; }

usage() {
  echo "Usage:"
  echo "  sudo bash switch-domain.sh check  NEW_DOMAIN          read-only report, changes nothing"
  echo "  sudo bash switch-domain.sh cert   NEW_DOMAIN [EMAIL]  get the SSL certificate (node keeps running)"
  echo "  sudo bash switch-domain.sh apply  NEW_DOMAIN          switch the node (only after Redbelly support confirms)"
  echo "  sudo bash switch-domain.sh status NEW_DOMAIN          service, certificate and sync status"
  exit 1
}

# ---------- domain names ----------

normalize_domain() {
  local d=${1,,}
  d=${d#http://}
  d=${d#https://}
  d=${d%%/*}
  d=${d%%:*}
  d=${d%.}
  printf '%s' "$d"
}

valid_domain() {
  local re='^([a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?[.])+[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$'
  [[ $1 =~ $re ]] && ! [[ $1 =~ ^[0-9.]+$ ]]
}

# ---------- certificates ----------

cert_matches() { # file domain
  local out
  out=$(openssl x509 -in "$1" -noout -checkhost "$2" 2>/dev/null)
  [[ $out == *"does match"* ]]
}
cert_valid() { openssl x509 -in "$1" -noout -checkend "${2:-0}" >/dev/null 2>&1; }
cert_end()   { openssl x509 -in "$1" -noout -enddate 2>/dev/null | cut -d= -f2; }
cert_cn()    { openssl x509 -in "$1" -noout -subject -nameopt RFC2253 2>/dev/null | awk -F'CN=' 'NF > 1 { print $2 }' | cut -d, -f1; }
key_matches() { # cert key
  local a b
  a=$(openssl x509 -in "$1" -noout -pubkey 2>/dev/null)
  b=$(openssl pkey -in "$2" -pubout 2>/dev/null)
  [ -n "$a" ] && [ "$a" = "$b" ]
}

can_read_as() { # user file
  if [ "$1" = root ]; then [ -r "$2" ]; return; fi
  if have runuser; then runuser -u "$1" -- test -r "$2" 2>/dev/null; return; fi
  if have sudo; then sudo -n -u "$1" test -r "$2" 2>/dev/null; return; fi
  return 1
}

# ---------- replacing the old domain in files ----------

# Replaces OLD_X with NEW_X only where it is a whole domain name, never inside a
# longer name such as sub.old-domain.xyz or myold-domain.xyz. Exit 0 = changed.
REPLACE_AWK='
BEGIN { o = ENVIRON["OLD_X"]; n = ENVIRON["NEW_X"]; lo = length(o); changed = 0 }
{
  s = $0; out = ""
  while (lo > 0 && (i = index(s, o)) > 0) {
    before = out substr(s, 1, i - 1)
    pc = substr(before, length(before), 1)
    nc = substr(s, i + lo, 1)
    if (pc !~ /[A-Za-z0-9.-]/ && nc !~ /[A-Za-z0-9-]/) { out = before n; changed = 1 }
    else out = before o
    s = substr(s, i + lo)
  }
  print out s
}
END { if (changed) exit 0; exit 3 }'

replace_in_string() { printf '%s' "$1" | OLD_X=$OLD NEW_X=$NEW awk "$REPLACE_AWK"; }

would_change() { # file
  [ -n "$OLD" ] || return 1
  grep -qIF -- "$OLD" "$1" 2>/dev/null || return 1
  OLD_X=$OLD NEW_X=$NEW awk "$REPLACE_AWK" "$1" >/dev/null 2>&1
}

# ---------- reading the service file ----------

# Value of --NAME=value or --NAME value; quotes removed; last one wins.
FLAG_AWK='
$1 ~ /^[#;]/ { next }
{
  for (i = 1; i <= NF; i++) {
    t = $i; gsub(q, "", t); gsub(dq, "", t)
    if (index(t, name "=") == 1) v = substr(t, length(name) + 2)
    else if (t == name && i < NF) { v = $(i + 1); gsub(q, "", v); gsub(dq, "", v) }
  }
}
END { if (v != "") print v }'

flag_value() { cat "${UNIT_FILES[@]}" 2>/dev/null | awk -v name="--$1" -v q="'" -v dq='"' "$FLAG_AWK"; }

unit_log() { # StandardOutput or StandardError
  cat "${UNIT_FILES[@]}" 2>/dev/null | grep -E "^[[:space:]]*$1=(append|file|truncate):" | tail -n 1 | cut -d: -f2- | awk '{ print $1 }'
}

read_setup() {
  UNIT=$(systemctl show -p FragmentPath --value "$SERVICE" 2>/dev/null)
  if [ -z "$UNIT" ] || [ ! -f "$UNIT" ]; then
    bad "$SERVICE was not found on this server"
    note "This script is for nodes installed with the official Redbelly installer (rbn-installer.run)."
    return 1
  fi
  UNIT_FILES=("$UNIT")
  local f
  for f in $(systemctl show -p DropInPaths --value "$SERVICE" 2>/dev/null); do
    [ -f "$f" ] && UNIT_FILES+=("$f")
  done
  SVC_USER=$(systemctl show -p User --value "$SERVICE" 2>/dev/null)
  SVC_USER=${SVC_USER:-root}
  SVC_GROUP=$(systemctl show -p Group --value "$SERVICE" 2>/dev/null)
  [ -n "$SVC_GROUP" ] || SVC_GROUP=$(id -gn "$SVC_USER" 2>/dev/null || echo root)
  CUR_CERT=$(flag_value tls.cert)
  CUR_KEY=$(flag_value tls.key)
  NODE_ID=$(flag_value nodeId)
  DATADIR=$(flag_value datadir)
  LOG=$(unit_log StandardOutput)
  LOG=${LOG:-$DEFAULT_LOG}
  ERRLOG=$(unit_log StandardError)
  ERRLOG=${ERRLOG:-${DEFAULT_LOG%.log}_error.log}
  CUR_CN=""
  if [ -n "$CUR_CERT" ] && [ -r "$CUR_CERT" ]; then CUR_CN=$(cert_cn "$CUR_CERT"); fi
  return 0
}

show_setup() {
  ok "Service file: $UNIT"
  note "Node ID ${NODE_ID:-unknown}, runs as user $SVC_USER"
  note "Chain data: ${DATADIR:-unknown} (this script never touches it)"
  if [ -z "$CUR_CERT" ]; then
    bad "The service file has no --tls.cert setting"
  elif [ -n "$CUR_CN" ]; then
    ok "Current certificate: $CUR_CN, valid until $(cert_end "$CUR_CERT")"
    note "$CUR_CERT"
  else
    warn "The certificate file the service file points to is missing: $CUR_CERT"
  fi
}

# ---------- DNS ----------

PUBLIC_IP_CACHE=""
public_ip() {
  local url ip
  [ -n "$PUBLIC_IP_CACHE" ] && { printf '%s' "$PUBLIC_IP_CACHE"; return 0; }
  for url in https://api.ipify.org https://ifconfig.me https://icanhazip.com; do
    ip=$(curl -4 -fsS --max-time 6 "$url" 2>/dev/null | head -n 1 | tr -d ' ')
    if [[ $ip =~ ^[0-9]+[.][0-9]+[.][0-9]+[.][0-9]+$ ]]; then
      PUBLIC_IP_CACHE=$ip
      printf '%s' "$ip"
      return 0
    fi
  done
  return 1
}

SERVER_IP=""
check_dns() { # domain
  local d=$1 a4 a6 mine pub="" ip good="" other="" rest="" v6other=""
  a4=$(getent ahostsv4 "$d" 2>/dev/null | awk '{ print $1 }' | sort -u)
  a6=$(getent ahostsv6 "$d" 2>/dev/null | awk '{ print $1 }' | grep -v '^::ffff:' | sort -u)
  mine=" $(hostname -I 2>/dev/null) "
  if [ -z "$a4" ]; then
    bad "$d has no A record yet, or DNS has not updated yet"
    note "Create an A record for $d that points to this server, wait a few minutes, then run this again."
    return
  fi
  for ip in $a4; do
    if [[ $mine == *" $ip "* ]]; then good="$good $ip"; else other="$other $ip"; fi
  done
  if [ -n "$other" ]; then
    pub=$(public_ip)
    for ip in $other; do
      if [ -n "$pub" ] && [ "$ip" = "$pub" ]; then good="$good $ip"; else rest="$rest $ip"; fi
    done
    other=$rest
  fi
  good=${good# }
  other=${other# }
  if [ -z "$other" ]; then
    ok "$d points to this server ($good)"
    SERVER_IP=${good%% *}
  elif [ -z "$good" ] && [ -z "$pub" ]; then
    warn "$d points to $other, but this script could not find this server's public IP to compare"
    note "Make sure $other is the IP shown in your VPS provider's panel."
    SERVER_IP=${other%% *}
  elif [ -z "$good" ]; then
    bad "$d points to $other, but this server is ${pub:-${mine// /}}"
    note "Change the A record for $d to this server's IP, wait a few minutes, then run this again."
  else
    bad "$d has several A records: $good (this server) and $other (not this server)"
    note "Delete the A records that don't point to this server."
  fi
  for ip in $a6; do
    [[ $mine == *" $ip "* ]] || v6other="$v6other $ip"
  done
  if [ -n "$v6other" ]; then
    warn "$d also has an AAAA (IPv6) record that is not on this server:$v6other"
    note "Delete it. Let's Encrypt and other nodes may use the IPv6 address."
  fi
}

# ---------- the new Let's Encrypt certificate ----------

NEW_CERT_OK=0
check_new_cert() { # mode: check | apply
  NEW_LE_CERT=$LE/live/$NEW/fullchain.pem
  NEW_LE_KEY=$LE/live/$NEW/privkey.pem
  NEW_CERT_OK=0
  if [ ! -f "$NEW_LE_CERT" ] || [ ! -f "$NEW_LE_KEY" ]; then
    if [ "$1" = check ]; then todo "No SSL certificate for $NEW yet"; else bad "No SSL certificate for $NEW yet"; fi
    note "Get it with: sudo bash switch-domain.sh cert $NEW"
    return
  fi
  if ! cert_matches "$NEW_LE_CERT" "$NEW"; then
    bad "The certificate in $LE/live/$NEW is not valid for $NEW"
    return
  fi
  if ! cert_valid "$NEW_LE_CERT" 0; then
    bad "The certificate for $NEW expired on $(cert_end "$NEW_LE_CERT")"
    note "Renew it with: sudo certbot renew --cert-name $NEW"
    return
  fi
  if ! key_matches "$NEW_LE_CERT" "$NEW_LE_KEY"; then
    bad "The private key in $LE/live/$NEW does not belong to its certificate"
    return
  fi
  ok "SSL certificate for $NEW is ready, valid until $(cert_end "$NEW_LE_CERT")"
  NEW_CERT_OK=1
}

# ---------- planning the switch ----------

other_cert_dir() { # the only other domain folder in /etc/redbelly/certs, if there is exactly one
  local d name found="" n=0
  for d in "$CERTS_ROOT"/*/; do
    [ -d "$d" ] || continue
    name=$(basename "$d")
    [ "$name" = "$NEW" ] && continue
    valid_domain "$name" || continue
    found=$name
    n=$((n + 1))
  done
  [ "$n" -eq 1 ] && printf '%s' "$found"
}

candidate_files() {
  local f
  for f in "${UNIT_FILES[@]}"; do readlink -f "$f"; done
  [ -d "$RB_ETC" ] && find "$RB_ETC" -path "$CERTS_ROOT" -prune -o -type f -print 2>/dev/null
  [ -d "$LE/renewal-hooks" ] && find "$LE/renewal-hooks" -type f -print 2>/dev/null
}

renewal_covered() {
  local f tdir
  tdir=$(dirname "$TARGET_CERT")
  [ "$LE_DIRECT" -eq 1 ] && return 0
  [ -d "$LE/renewal-hooks" ] || return 1
  while IFS= read -r f; do
    if OLD_X=$OLD NEW_X=$NEW awk "$REPLACE_AWK" "$f" 2>/dev/null | grep -qF -- "$tdir"; then return 0; fi
  done < <(find "$LE/renewal-hooks" -type f 2>/dev/null)
  return 1
}

plan_switch() {
  OLD=""
  LE_DIRECT=0
  NEED_COPY=0
  EDIT_FILES=()
  REF_DIR=""
  REF_CERT=""
  REF_KEY=""
  TARGET_CERT=""
  TARGET_KEY=""
  if [ -z "$CUR_CERT" ] || [ -z "$CUR_KEY" ]; then
    bad "The service file has no --tls.cert or --tls.key setting"
    return 1
  fi
  if [[ $CUR_CERT != /* ]] || [[ $CUR_KEY != /* ]] || [[ $CUR_CERT$CUR_KEY == *'$'* ]]; then
    bad "This script can't read the certificate paths in the service file: $CUR_CERT"
    return 1
  fi

  # Which domain to replace
  local dirname_cur
  dirname_cur=$(basename "$(dirname "$CUR_CERT")")
  if [ -n "$CUR_CN" ] && [ "$CUR_CN" != "$NEW" ]; then
    OLD=$CUR_CN
  elif [ -z "$CUR_CN" ] && [ "$dirname_cur" != "$NEW" ] && valid_domain "$dirname_cur"; then
    OLD=$dirname_cur
  else
    OLD=$(other_cert_dir)
  fi

  # Where the node will read the new certificate from
  if [ -n "$OLD" ]; then
    TARGET_CERT=$(replace_in_string "$CUR_CERT")
    TARGET_KEY=$(replace_in_string "$CUR_KEY")
  else
    TARGET_CERT=$CUR_CERT
    TARGET_KEY=$CUR_KEY
  fi
  if [ "$TARGET_CERT" = "$TARGET_KEY" ]; then
    bad "The service file uses the same file for the certificate and the key"
    return 1
  fi
  case "$TARGET_CERT" in
    "$LE"/*)
      if [ "$(dirname "$TARGET_CERT")" = "$LE/live/$NEW" ] && [ "$(dirname "$TARGET_KEY")" = "$LE/live/$NEW" ]; then
        LE_DIRECT=1
      else
        bad "The node reads its certificate from $(dirname "$TARGET_CERT"), which this script can't update"
        note "See 'Other setups' in the guide."
        return 1
      fi
      ;;
  esac

  # Files that will change
  local f
  if [ -n "$OLD" ]; then
    while IFS= read -r f; do
      [ -n "$f" ] || continue
      if would_change "$f"; then EDIT_FILES+=("$f"); fi
    done < <(candidate_files | sort -u)
  fi

  # Certificate files to install, and owner/permissions to copy
  if [ "$LE_DIRECT" -eq 0 ]; then
    if [ -f "$TARGET_CERT" ] && [ -f "$TARGET_KEY" ] && [ "$NEW_CERT_OK" -eq 1 ] && cmp -s "$NEW_LE_CERT" "$TARGET_CERT" && cmp -s "$NEW_LE_KEY" "$TARGET_KEY"; then
      NEED_COPY=0
    else
      NEED_COPY=1
    fi
    [ -f "$CUR_CERT" ] && REF_CERT=$CUR_CERT
    [ -f "$CUR_KEY" ] && REF_KEY=$CUR_KEY
    [ -d "$(dirname "$CUR_CERT")" ] && REF_DIR=$(dirname "$CUR_CERT")
    if [ -n "$OLD" ] && [ -d "$CERTS_ROOT/$OLD" ]; then
      [ -n "$REF_CERT" ] || { [ -f "$CERTS_ROOT/$OLD/$(basename "$TARGET_CERT")" ] && REF_CERT=$CERTS_ROOT/$OLD/$(basename "$TARGET_CERT"); }
      [ -n "$REF_KEY" ] || { [ -f "$CERTS_ROOT/$OLD/$(basename "$TARGET_KEY")" ] && REF_KEY=$CERTS_ROOT/$OLD/$(basename "$TARGET_KEY"); }
      [ -n "$REF_DIR" ] || REF_DIR=$CERTS_ROOT/$OLD
    fi
  fi
  return 0
}

nothing_to_change() { [ "$NEED_COPY" -eq 0 ] && [ ${#EDIT_FILES[@]} -eq 0 ]; }

show_plan() {
  if nothing_to_change; then
    ok "The service file already uses the certificate for $NEW"
    return
  fi
  [ -n "$OLD" ] && note "Old domain: $OLD"
  if [ "$LE_DIRECT" -eq 1 ]; then
    note "The node will read the certificate straight from $LE/live/$NEW"
    if [ "$NEW_CERT_OK" -eq 1 ] && ! can_read_as "$SVC_USER" "$TARGET_KEY"; then
      bad "User $SVC_USER can't read $TARGET_KEY"
      note "See 'Other setups' in the guide."
    fi
  elif [ "$NEED_COPY" -eq 1 ]; then
    note "New certificate goes to: $TARGET_CERT"
    note "New key goes to:         $TARGET_KEY"
  fi
  if [ ${#EDIT_FILES[@]} -gt 0 ]; then
    note "Files where $OLD becomes $NEW:"
    local f
    for f in "${EDIT_FILES[@]}"; do note "  $f"; done
  fi
  if renewal_covered; then
    ok "Certificate renewals will keep the node's copy up to date"
  else
    warn "No certbot renewal hook copies renewed certificates to $(dirname "$TARGET_CERT")"
    note "Let's Encrypt certificates last 90 days. After each renewal, run this script's apply command again."
  fi
}

find_leftovers() { # folders named " that the first version of the guide created by mistake
  find $HOME_DIRS -maxdepth 2 -type d -name '"' 2>/dev/null | sort -u
}

show_leftovers() {
  local d
  while IFS= read -r d; do
    [ -n "$d" ] || continue
    warn "Leftover folder from the first version of this guide: $d"
    note "It holds a copy of your certificate and key. Delete it with: sudo rm -r '$d'"
  done < <(find_leftovers)
}

# ---------- node log and sync ----------

# Log lines with the node's block: "current local block N" and "Sync request is sent to downloader, ID: X, from N"
BLOCK_RE='current local block [0-9]+|downloader, ID: [0-9]+, from [0-9]+'
local_block() { tail -n 20000 "$LOG" 2>/dev/null | grep -oE "$BLOCK_RE" | tail -n 1 | awk '{ print $NF }'; }
local_block_time() { tail -n 20000 "$LOG" 2>/dev/null | grep -E "$BLOCK_RE" | tail -n 1 | grep -oE 'time=[^ ]+' | cut -d= -f2; }

network_block() {
  local out hex
  out=$(curl -fsS --max-time 10 -H 'Content-Type: application/json' --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' "$RPC_URL" 2>/dev/null) || return 1
  hex=$(printf '%s' "$out" | grep -oE '"result" *: *"0x[0-9a-fA-F]+"' | grep -oE '0x[0-9a-fA-F]+')
  [ -n "$hex" ] && printf '%d' "$hex"
}

served_cert_cn() {
  local pem
  pem=$(timeout 8 openssl s_client -connect "127.0.0.1:$TLS_PORT" -servername "$NEW" -alpn h2 < /dev/null 2>/dev/null)
  printf '%s' "$pem" | openssl x509 -noout -subject -nameopt RFC2253 2>/dev/null | awk -F'CN=' 'NF > 1 { print $2 }' | cut -d, -f1
}

# "received 0 responses" lines since the node last started (within the last 2000 log lines)
zero_responses() { tail -n 2000 "$LOG" 2>/dev/null | awk '/SEVM Node Started/ { n = 0 } /received 0 responses/ { n++ } END { print n + 0 }'; }

show_sync() {
  local lb lt nb age="" ts
  lb=$(local_block)
  lt=$(local_block_time)
  nb=$(network_block)
  if [ -n "$lt" ]; then
    ts=$(date -d "$lt" +%s 2>/dev/null)
    [ -n "$ts" ] && age=$(( ($(date +%s) - ts) / 60 ))
  fi
  if [ -z "$lb" ]; then
    warn "No block number found in the node log yet ($LOG)"
  else
    note "Node block (from the log): $lb${age:+, logged $age min ago}"
  fi
  if [ -z "$nb" ]; then
    warn "Could not read the network block from $RPC_URL"
  else
    note "Network block:             $nb"
  fi
  if [ -n "$lb" ] && [ -n "$nb" ]; then
    if [ $((nb - lb)) -le 3 ]; then ok "The node is in sync with the network"
    else warn "The node is $((nb - lb)) blocks behind. Run status again in a few minutes; the number should go up."; fi
  fi
  if [ -n "$age" ] && [ "$age" -gt 15 ]; then
    warn "The node has not logged a new block for $age minutes"
  fi
  local z
  z=$(zero_responses)
  if [ "${z:-0}" -gt 0 ]; then
    warn "The node is not getting answers from other nodes ('received 0 responses' $z times since it last started)"
    note "Before the switch: if support has already changed your hostname, this is expected until you run apply."
    note "After the switch: wait a few minutes. If it continues, ask Redbelly support to check your hostname."
  fi
}

# ---------- support ticket ----------

print_ticket() {
  local old=${OLD:-<OLD_DOMAIN>} id=${NODE_ID:-<YOUR_NODE_ID>} ip=${SERVER_IP:-<YOUR_VPS_IP>}
  echo
  echo "------------- Support ticket text (copy everything between the lines) -------------"
  echo "Subject: Hostname change for Node ID $id"
  echo
  echo "Hi team,"
  echo
  echo "My node (Node ID: $id) is registered with hostname $old."
  echo "That domain has expired, so I have moved to $NEW, which points to the"
  echo "same server IP $ip. Ports, vote/signing address and reward address"
  echo "are unchanged."
  echo
  echo "Please update the registered hostname for my node to $NEW and let me"
  echo "know when it is done, so I can switch my node to the new certificate."
  echo "The switch needs a restart of about 1 minute."
  echo
  echo "Thanks"
  echo "-----------------------------------------------------------------------------------"
}

# ---------- changing files ----------

make_backup() {
  BACKUP=$BACKUP_ROOT/rbn-backup-$(date +%Y%m%d-%H%M%S)
  mkdir -p "$BACKUP" || return 1
  chmod 700 "$BACKUP"
  local list=() f
  for f in "${UNIT_FILES[@]}"; do list+=("$(readlink -f "$f")"); done
  [ ${#EDIT_FILES[@]} -gt 0 ] && list+=("${EDIT_FILES[@]}")
  [ -d "$RB_ETC" ] && list+=("$RB_ETC")
  [ -d "$LE/renewal-hooks" ] && list+=("$LE/renewal-hooks")
  for f in "$TARGET_CERT" "$TARGET_KEY"; do [ -e "$f" ] && list+=("$f"); done
  BACKUP_ITEMS=()
  # Unique paths, without files that are already inside a folder on the list
  while IFS= read -r f; do BACKUP_ITEMS+=("$f"); done < <(for f in "${list[@]}"; do echo "$f"; done | LC_ALL=C sort -u | awk '{ for (i = 1; i <= n; i++) if (index($0, k[i] "/") == 1) next; k[++n] = $0; print }')
  cp -a --parents "${BACKUP_ITEMS[@]}" "$BACKUP"/ || return 1
  # Each item is copied back to its own folder. Never copy the backup folder itself onto /.
  {
    echo "#!/bin/bash"
    echo "# Written by switch-domain.sh: puts back the files saved in $BACKUP, then restarts the node."
    for f in "${BACKUP_ITEMS[@]}"; do echo "cp -a '$BACKUP$f' '$(dirname "$f")/'"; done
    echo "systemctl daemon-reload"
    echo "systemctl restart $SERVICE"
    echo "echo 'Restored the files from $BACKUP and restarted $SERVICE.'"
  } > "$BACKUP/restore.sh"
}

restore_backup() {
  local f ok=1
  for f in "${BACKUP_ITEMS[@]}"; do cp -a "$BACKUP$f" "$(dirname "$f")/" || ok=0; done
  if [ "$ok" -eq 1 ]; then
    echo "  Restored all files from $BACKUP. The node was not restarted and runs as before."
  else
    echo "  Could not restore every file. Run: sudo bash $BACKUP/restore.sh"
  fi
}

put_file() { # source target reference fallback_mode
  local tmp="$2.switch-domain-tmp"
  cp -L "$1" "$tmp" || return 1
  if [ -n "$3" ] && [ -e "$3" ]; then
    chown --reference="$3" "$tmp" && chmod --reference="$3" "$tmp" || return 1
  else
    chown "root:$SVC_GROUP" "$tmp" && chmod "$4" "$tmp" || return 1
  fi
  mv -f "$tmp" "$2"
}

install_cert_files() {
  local d
  for d in "$(dirname "$TARGET_CERT")" "$(dirname "$TARGET_KEY")"; do
    [ -d "$d" ] && continue
    mkdir -p "$d" || return 1
    if [ -n "$REF_DIR" ] && [ -d "$REF_DIR" ]; then
      chown --reference="$REF_DIR" "$d" && chmod --reference="$REF_DIR" "$d" || return 1
    else
      chown "root:$SVC_GROUP" "$d" && chmod 750 "$d" || return 1
    fi
  done
  put_file "$NEW_LE_CERT" "$TARGET_CERT" "$REF_CERT" 644 || return 1
  put_file "$NEW_LE_KEY" "$TARGET_KEY" "$REF_KEY" 640 || return 1
  ok "Installed the new certificate in $(dirname "$TARGET_CERT")"
}

edit_files() {
  local f tmp rc
  for f in "${EDIT_FILES[@]}"; do
    tmp=$(mktemp) || return 1
    OLD_X=$OLD NEW_X=$NEW awk "$REPLACE_AWK" "$f" > "$tmp"
    rc=$?
    if [ "$rc" -ne 0 ] && [ "$rc" -ne 3 ]; then rm -f "$tmp"; return 1; fi
    cat "$tmp" > "$f" || { rm -f "$tmp"; return 1; }
    rm -f "$tmp"
    ok "Updated $f"
  done
}

verify_switch() {
  local c k f
  c=$(flag_value tls.cert)
  k=$(flag_value tls.key)
  if [ "$c" != "$TARGET_CERT" ] || [ "$k" != "$TARGET_KEY" ]; then
    bad "The service file does not point to the new certificate ($c)"
    return 1
  fi
  if ! cert_matches "$c" "$NEW" || ! key_matches "$c" "$k"; then
    bad "The certificate or key at the new location is not right"
    return 1
  fi
  if ! can_read_as "$SVC_USER" "$c" || ! can_read_as "$SVC_USER" "$k"; then
    bad "User $SVC_USER can't read the new certificate or key"
    return 1
  fi
  if [ ${#EDIT_FILES[@]} -gt 0 ]; then
    for f in "${EDIT_FILES[@]}"; do
      if would_change "$f"; then bad "The old domain is still in $f"; return 1; fi
    done
  fi
  ok "Checked: the service file points to the certificate for $NEW and user $SVC_USER can read it"
}

restart_node() {
  local size blk="" state
  size=$(stat -c %s "$LOG" 2>/dev/null || echo 0)
  systemctl daemon-reload
  systemctl reset-failed "$SERVICE" >/dev/null 2>&1
  if ! systemctl restart "$SERVICE"; then
    bad "systemctl restart $SERVICE failed"
    return 1
  fi
  printf '  Restarted. Waiting for the node to start (up to 3 minutes) '
  for _ in $(seq 1 36); do
    sleep 5
    printf '.'
    if systemctl is-failed --quiet "$SERVICE"; then break; fi
    blk=$(tail -c +$((size + 1)) "$LOG" 2>/dev/null | grep -oE "$BLOCK_RE" | tail -n 1 | awk '{ print $NF }')
    [ -n "$blk" ] && break
  done
  echo
  state=$(systemctl is-active "$SERVICE" 2>/dev/null)
  if [ -n "$blk" ] && [ "$blk" -ge 1000 ]; then
    ok "The node started and continues from block $blk (no resync)"
  elif [ -n "$blk" ]; then
    warn "The node started from block $blk, near the start of the chain"
    note "This script never touches the chain data. If the node was synced before, ask Redbelly support before you do anything else."
  elif [ "$state" = active ]; then
    warn "The node is running but has not logged a block number yet"
    note "Check again in a few minutes with: sudo bash switch-domain.sh status $NEW"
  else
    bad "The node is not running (state: ${state:-unknown})"
    note "Last lines of the error log ($ERRLOG):"
    tail -n 15 "$ERRLOG" 2>/dev/null | sed 's/^/          /'
    note "More details: sudo journalctl -u $SERVICE -n 50 --no-pager"
    note "To undo the switch: sudo bash $BACKUP/restore.sh"
    return 1
  fi
}

confirm() {
  local ans=""
  echo
  echo "Redbelly support must already have confirmed that your node's hostname is now $NEW."
  read -r -p "Type yes to switch the node now (it restarts for about 1 minute): " ans < /dev/tty
  [ "$ans" = yes ]
}

# ---------- commands ----------

cmd_check() {
  section "Node setup"
  read_setup || die "Nothing to check."
  show_setup
  section "New domain: $NEW"
  check_dns "$NEW"
  check_new_cert check
  section "Switch plan"
  plan_switch && show_plan
  section "Node log"
  show_sync
  show_leftovers
  section "Result"
  if [ "$FAILS" -gt 0 ]; then
    echo "  Fix the FAIL items above, then run this check again."
  elif [ "$NEW_CERT_OK" -eq 0 ]; then
    echo "  Next: get the SSL certificate (the node keeps running):"
    echo "    sudo bash switch-domain.sh cert $NEW"
  elif nothing_to_change; then
    echo "  This node is already set up for $NEW. To see if it is syncing:"
    echo "    sudo bash switch-domain.sh status $NEW"
  else
    print_ticket
    echo
    echo "  Next: send the ticket above to Redbelly support. As soon as they confirm the change, run:"
    echo "    sudo bash switch-domain.sh apply $NEW"
  fi
}

cmd_cert() {
  if [ -n "$EMAIL" ] && [[ $EMAIL != *@*.* ]]; then die "'$EMAIL' is not an email address."; fi
  section "SSL certificate for $NEW"
  NEW_LE_CERT=$LE/live/$NEW/fullchain.pem
  NEW_LE_KEY=$LE/live/$NEW/privkey.pem
  check_dns "$NEW"
  [ "$FAILS" -eq 0 ] || die "Fix DNS first. Let's Encrypt blocks new attempts for an hour after 5 failures."
  if [ -f "$NEW_LE_CERT" ] && cert_matches "$NEW_LE_CERT" "$NEW" && cert_valid "$NEW_LE_CERT" 2592000 && key_matches "$NEW_LE_CERT" "$NEW_LE_KEY"; then
    ok "You already have a certificate for $NEW, valid until $(cert_end "$NEW_LE_CERT")"
  else
    have certbot || die "certbot is not installed. Install it with: sudo apt update && sudo apt install -y certbot"

    local line proc names hook_stop="" hook_start="" svc=""
    line=""
    have ss && line=$(ss -Hltnp 'sport = :80' 2>/dev/null | head -n 1)
    if [ -z "$line" ]; then
      ok "Port 80 is free"
    else
      proc=$(printf '%s' "$line" | grep -oE '[(][(]"[^"]+"' | head -n 1 | cut -d'"' -f2)
      case "$proc" in
        docker-proxy|docker)
          names=$(docker ps --filter publish=80 --format '{{.Names}}' 2>/dev/null)
          if [ -n "$names" ] && [ "$(printf '%s' "$names" | grep -c .)" -eq 1 ]; then
            hook_stop="docker stop $names"
            hook_start="docker start $names"
            ok "Port 80 is used by Docker container $names. It will be stopped for a few seconds."
          else
            die "Port 80 is used by Docker. Stop that container, run this command again, then start the container."
          fi
          ;;
        "")
          die "Something is using port 80 and the script can't tell what. Free port 80, then run this again."
          ;;
        *)
          if systemctl is-active --quiet "$proc"; then
            svc=$proc
            hook_stop="systemctl stop $proc"
            hook_start="systemctl start $proc"
            ok "Port 80 is used by $proc. Certbot will stop it for a few seconds and start it again, also at every renewal."
          else
            die "Port 80 is used by $proc, which this script can't stop. Stop it, run this again, then start it."
          fi
          ;;
      esac
    fi
    if have ufw && ufw status 2>/dev/null | head -n 1 | grep -q 'Status: active'; then
      if ufw status 2>/dev/null | grep -qE '^80(/tcp)?[[:space:]].*ALLOW'; then
        ok "The firewall (ufw) allows port 80"
      else
        warn "The firewall (ufw) is active and may block port 80. If certbot times out, run: sudo ufw allow 80"
      fi
    fi

    local args=(certonly --standalone -d "$NEW" --non-interactive --agree-tos --keep-until-expiring)
    if [ -n "$EMAIL" ]; then args+=(-m "$EMAIL"); else args+=(--register-unsafely-without-email); fi
    if [ -n "$hook_stop" ]; then args+=(--pre-hook "$hook_stop" --post-hook "$hook_start"); fi
    echo
    echo "  Running certbot..."
    certbot "${args[@]}"
    echo
    if [ -n "$svc" ] && ! systemctl is-active --quiet "$svc"; then
      warn "$svc did not start again after certbot. This does not affect the Redbelly node."
      note "See why with: sudo journalctl -u $svc -n 30 --no-pager"
    fi
    check_new_cert apply
    if [ "$NEW_CERT_OK" -ne 1 ]; then
      note "Common causes: the domain does not point to this server yet, port 80 is closed in a"
      note "firewall (ufw or your VPS provider's firewall), or a Cloudflare proxy (orange cloud) is on."
      die "No certificate yet. Fix the cause above and run this command again."
    fi
  fi

  FAILS=0
  if read_setup >/dev/null; then
    if [ -n "$CUR_CN" ] && [ "$CUR_CN" != "$NEW" ]; then OLD=$CUR_CN; else OLD=""; fi
  fi
  print_ticket
  echo
  echo "  Next: send the ticket above to Redbelly support. As soon as they confirm the change, run:"
  echo "    sudo bash switch-domain.sh apply $NEW"
}

cmd_apply() {
  section "Node setup"
  read_setup || die "Nothing was changed."
  show_setup
  section "New domain: $NEW"
  check_dns "$NEW"
  check_new_cert apply
  section "Switch plan"
  plan_switch && show_plan
  [ "$FAILS" -eq 0 ] || die "Nothing was changed. Fix the FAIL items above, then run apply again."

  if nothing_to_change; then
    if systemctl is-active --quiet "$SERVICE" && [ "$(served_cert_cn)" = "$NEW" ]; then
      section "Result"
      ok "Nothing to change: the node already runs with the certificate for $NEW"
      section "Sync"
      show_sync
      return
    fi
    note "The files are already updated, but the running node still needs a restart."
  fi

  confirm || die "Cancelled. Nothing was changed."
  section "Switching"
  make_backup || die "Could not create the backup. Nothing was changed."
  ok "Backup of the current files: $BACKUP"
  if [ "$NEED_COPY" -eq 1 ]; then
    install_cert_files || { restore_backup; die "Could not install the certificate."; }
  fi
  edit_files || { restore_backup; die "Could not update the files."; }
  verify_switch || { restore_backup; die "The check after the changes failed."; }
  restart_node || die "The node did not start. See above."
  section "Sync"
  show_sync
  section "Next steps"
  echo "  1. Check again in 10-15 minutes: sudo bash switch-domain.sh status $NEW"
  if [ -n "$OLD" ] && [ -d "$LE/live/$OLD" ]; then
    echo "  2. When the node is syncing, delete the old certificate: sudo certbot delete --cert-name $OLD"
  fi
  echo "  To undo the switch: sudo bash $BACKUP/restore.sh"
  echo "  The backup folder contains a copy of your keys. Delete it after a few days: sudo rm -r $BACKUP"
  show_leftovers
}

cmd_status() {
  section "Node status"
  read_setup || die "Nothing to check."
  local state since served
  state=$(systemctl is-active "$SERVICE" 2>/dev/null)
  if [ "$state" = active ]; then
    since=$(systemctl show -p ActiveEnterTimestamp --value "$SERVICE" 2>/dev/null)
    ok "$SERVICE is running${since:+ (since $since)}"
  else
    bad "$SERVICE is not running (state: ${state:-unknown})"
    note "Last lines of the error log ($ERRLOG):"
    tail -n 10 "$ERRLOG" 2>/dev/null | sed 's/^/          /'
  fi
  if [ "$CUR_CN" = "$NEW" ]; then
    ok "The service file uses the certificate for $NEW, valid until $(cert_end "$CUR_CERT")"
  else
    bad "The service file uses the certificate for ${CUR_CN:-a missing file}, not $NEW"
    note "Run: sudo bash switch-domain.sh apply $NEW"
  fi
  if [ "$state" = active ]; then
    served=$(served_cert_cn)
    if [ "$served" = "$NEW" ]; then ok "The node serves the certificate for $NEW on port $TLS_PORT"
    elif [ -n "$served" ]; then bad "The node still serves the certificate for $served on port $TLS_PORT"; note "Run: sudo bash switch-domain.sh apply $NEW"
    else warn "Could not read the certificate on port $TLS_PORT (the node may still be starting)"; fi
  fi
  section "Sync"
  show_sync
  show_leftovers
}

# ---------- main ----------

CMD=${1:-}
NEW=$(normalize_domain "${2:-}")
EMAIL=${3:-}
case "$CMD" in check|cert|apply|status) ;; *) usage ;; esac
[ -n "$NEW" ] || usage
valid_domain "$NEW" || die "'${2:-}' is not a domain name. Use only the domain, like mynode.com"
[ "$(id -u)" -eq 0 ] || die "Run it with sudo: sudo bash switch-domain.sh $CMD $NEW"
have openssl || die "openssl is not installed. Install it with: sudo apt install -y openssl"
echo "switch-domain.sh $VERSION: $CMD $NEW"
"cmd_$CMD"
