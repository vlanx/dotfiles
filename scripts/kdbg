#!/usr/bin/env bash

kdbg() {
  local namespace="default"
  local node=""
  local serviceaccount="default"
  local privileged=0
  local host_network=0
  local image="${KDBG_IMAGE:-ghcr.io/vlanx/k8s-debug:latest}"
  local cmd=""
  local flag=""
  local value=""

  while [ "$#" -gt 0 ]; do
    flag="$1"
    value="${2:-}"

    case "$flag" in
    -n | --namespace)
      [ -n "$value" ] || {
        echo "kdbg: missing value for $flag" >&2
        return 1
      }
      namespace="$value"
      shift 2
      ;;
    -N | --node)
      [ -n "$value" ] || {
        echo "kdbg: missing value for $flag" >&2
        return 1
      }
      node="$value"
      shift 2
      ;;
    --serviceaccount)
      [ -n "$value" ] || {
        echo "kdbg: missing value for $flag" >&2
        return 1
      }
      serviceaccount="$value"
      shift 2
      ;;
    --cmd)
      [ -n "$value" ] || {
        echo "kdbg: missing value for $flag" >&2
        return 1
      }
      cmd="$value"
      shift 2
      ;;
    --privileged)
      privileged=1
      shift
      ;;
    --host-network)
      host_network=1
      shift
      ;;
    --help | -h)
      cat <<'EOF'
Usage: kdbg [options]

Options:
  -n, --namespace NAME       Namespace for the debug pod.
  -N, --node NAME            Pin the pod to a specific node.
      --serviceaccount NAME  ServiceAccount name (default: default).
      --privileged           Run the pod in privileged mode.
      --host-network         Join the node network namespace.
      --cmd 'COMMAND'        Run a one-shot command instead of an interactive shell.
  -h, --help                 Show this help.

Environment:
  KDBG_IMAGE                 Image to run (default: ghcr.io/vlanx/k8s-debug:latest).
EOF
      return 0
      ;;
    *)
      echo "kdbg: unknown argument: $flag" >&2
      return 1
      ;;
    esac
  done

  # Start of kubectl command building
  local -a kubectl_cmd
  kubectl_cmd=(kubectl)

  local -a run_args
  run_args=(run k8s-debug-pod --namespace "$namespace" --image "$image" --restart=Never --rm --attach)

  if [ "$privileged" -eq 1 ]; then
    run_args+=(--privileged)
  fi

  # Create overrides in case we want to deploy to a specific node, use a specific SA or need to be in the host network namespace
  if [ "$serviceaccount" != "default" ] || [ -n "$node" ] || [ "$host_network" -eq 1 ]; then
    local overrides='{"apiVersion":"v1","spec":{'
    local sep=""

    if [ "$serviceaccount" != "default" ]; then
      overrides+='"serviceAccountName":"'"$serviceaccount"'"'
      sep=","
    fi

    if [ -n "$node" ]; then
      overrides+="${sep}\"nodeName\":\"$node\""
      sep=","
    fi

    if [ "$host_network" -eq 1 ]; then
      overrides+="${sep}\"hostNetwork\":true,\"dnsPolicy\":\"ClusterFirstWithHostNet\""
    fi

    overrides+='}}'
    run_args+=(--overrides "$overrides")
  fi

  if [ -n "$cmd" ]; then
    run_args+=(--command -- /bin/sh -lc "$cmd")
  else
    run_args+=(-i -t --command -- /bin/sh)
  fi

  {
    printf 'kdbg: launching debug pod in namespace %s' "$namespace" >&2
    if [ -n "$node" ]; then
      printf ' pinned to node %s' "$node" >&2
    fi
    printf '\n' >&2
  }

  "${kubectl_cmd[@]}" "${run_args[@]}"
}
