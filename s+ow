#!/bin/sh

# surplus on wheels (s+ow) - a pure shell script to run surplus with mdtest using the termux-api
# ------------------------
# public domain, unlicence

# shellcheck disable=SC2059
LOCATION_FALLBACK="%d%d%d\nSingapore?"
# shellcheck disable=SC2269
LOCATION_PRIORITISE_NETWORK="$LOCATION_PRIORITISE_NETWORK"
LOCATION_TIMEOUT=${LOCATION_TIMEOUT:-50}

# shellcheck disable=SC2269
SPOW_TARGETS="$SPOW_TARGETS"
SPOW_CACHE_DIR="$HOME/.cache/s+ow"
SPOW_BRIDGES="$HOME/.s+ow-bridges"
SPOW_CRON=${SPOW_CRON:-n}

# per-tool session logs
SPOW_NETLC_OUT="$SPOW_CACHE_DIR/location.net.json"
SPOW_GPSLC_OUT="$SPOW_CACHE_DIR/location.gps.json"
SPOW_LOCTN_OUT="$SPOW_CACHE_DIR/location.json"
SPOW_SPLUS_OUT="$SPOW_CACHE_DIR/surplus.out.log"
SPOW_SPLUS_ERR="$SPOW_CACHE_DIR/surplus.err.log"

# per-session collated logs
SPOW_SESH_OUT="$SPOW_CACHE_DIR/out.log"
SPOW_SESH_ERR="$SPOW_CACHE_DIR/err.log"

# per-week collated logs
SPOW_WEEK_PRE="$SPOW_CACHE_DIR/$(date +%Y)W$(date +"%V")"
SPOW_WEEK_OUT="$SPOW_WEEK_PRE.out.log"
SPOW_WEEK_ERR="$SPOW_WEEK_PRE.err.log"

# last successful surplus output
SPOW_LAST_OUT="$SPOW_CACHE_DIR/last"

# message to be sent
SPOW_MESSAGE="$SPOW_CACHE_DIR/message"

# list of fakes
SPOW_FAKE_OUT="$SPOW_CACHE_DIR/fake"

# check for network location priority
if [ "$LOCATION_PRIORITISE_NETWORK" = "n" ]; then
	LOCATION_PRIORITISE_NETWORK=""
fi

# check for cron status
if [ "$SPOW_CRON" = "n" ]; then
	SPOW_CRON=""
fi

# ensure commands exist
if ! command -v termux-location >/dev/null 2>&1; then
	printf "s+ow: error: termux-location is not installed.\ninstall it with 'pkg install termux-api' and with installing the termux:api app from the play store or f-droid.\n"
	exit 1
fi

if ! command -v surplus >/dev/null 2>&1; then
	printf "s+ow: error: surplus is not installed.\ninstall it with 'pip install https://github.com/markjoshwel/surplus/releases/latest/download/surplus-latest-py3-none-any.whl'\n"
	exit 1
fi

# ensure directories
mkdir -p "$SPOW_CACHE_DIR"

# create new session logs
rm -f "$SPOW_LOCTN_OUT" "$SPOW_SPLUS_OUT" "$SPOW_SPLUS_ERR" \
	"$SPOW_SESH_OUT" "$SPOW_SESH_ERR"
touch "$SPOW_NETLC_OUT" "$SPOW_GPSLC_OUT" "$SPOW_LOCTN_OUT" \
	"$SPOW_SPLUS_OUT" "$SPOW_SPLUS_ERR" \
	"$SPOW_SESH_OUT" "$SPOW_SESH_ERR" \
	"$SPOW_WEEK_OUT" "$SPOW_WEEK_ERR" \
	"$SPOW_BRIDGES" "$SPOW_FAKE_OUT"

# 0 is nominal
# 1 is an termux-location error
# 2 is a surplus error
# 3 is a bridge/message send error
status=0

bridge_failures=0
bridge_returns=""

locate() {
	# spawn termux-location processes
	(
		termux-location -p "network" >"$SPOW_NETLC_OUT"
		if [ -s "$SPOW_NETLC_OUT" ]; then
			printf "net" | tee -a "$SPOW_SESH_ERR"
		else
			printf "net?" | tee -a "$SPOW_SESH_ERR"
		fi
		cat "$SPOW_NETLC_OUT" >>"$SPOW_SESH_OUT"
	) &
	tl_net_pid="$!"
	sleep 1
	(
		termux-location -p "gps" >"$SPOW_GPSLC_OUT"
		if [ -s "$SPOW_GPSLC_OUT" ]; then
			printf "gps" | tee -a "$SPOW_SESH_ERR"
		else
			printf "gps?" | tee -a "$SPOW_SESH_ERR"
		fi
		cat "$SPOW_GPSLC_OUT" >>"$SPOW_SESH_OUT"
	) &
	tl_gps_pid="$!"

	# wait until timeout or both finished
	printf "running termux-location" | tee -a "$SPOW_SESH_ERR"
	while [ "$LOCATION_TIMEOUT" -gt 0 ]; do
		# get process statuses
		kill -0 "$tl_net_pid" >/dev/null 2>&1
		tl_net_status="$?"
		kill -0 "$tl_gps_pid" >/dev/null 2>&1
		tl_gps_status="$?"

		# break if both finished
		if [ "$tl_net_status" -eq 1 ] && [ "$tl_gps_status" -eq 1 ]; then
			break
		fi

		# exception: if network is proritised: just use that
		if [ "$tl_net_status" -eq 1 ] && [ -n "$LOCATION_PRIORITISE_NETWORK" ]; then
			# break only if theres an actual response
			if [ -s "$SPOW_NETLC_OUT" ]; then
				break
			fi
			# else just keep on waiting for gps to finish
		fi

		sleep 1
		printf "." | tee -a "$SPOW_SESH_ERR"
		LOCATION_TIMEOUT=$((LOCATION_TIMEOUT - 1))
	done
	if [ "$LOCATION_TIMEOUT" -eq 0 ]; then
		printf " errored (timeout)\n" | tee -a "$SPOW_SESH_ERR"
	else
		printf " nominal\n" | tee -a "$SPOW_SESH_ERR"
	fi

	# check outputs
	printf "determining output: " | tee -a "$SPOW_SESH_ERR"
	if [ -s "$SPOW_NETLC_OUT" ] && [ -s "$SPOW_GPSLC_OUT" ]; then
		printf "both succeeded, "
		acc_net="$(grep "\"accuracy\"" <"$SPOW_NETLC_OUT" | awk -F ': ' '{print $2}' | tr -d ',')"
		acc_gps="$(grep "\"accuracy\"" <"$SPOW_GPSLC_OUT" | awk -F ': ' '{print $2}' | tr -d ',')"

		# compare accuracy
		if awk -v n1="$acc_net" -v n2="$acc_gps" 'BEGIN { if (n1 < n2) exit 0; else exit 1; }'; then
			printf "choosing network (%s < %s)" "$acc_net" "$acc_gps" | tee -a "$SPOW_SESH_ERR"
			cat "$SPOW_NETLC_OUT" >"$SPOW_LOCTN_OUT"
		else
			printf "choosing gps (%s < %s)" "$acc_gps" "$acc_net" | tee -a "$SPOW_SESH_ERR"
			cat "$SPOW_GPSLC_OUT" >"$SPOW_LOCTN_OUT"
		fi

		cat "$SPOW_GPSLC_OUT" >"$SPOW_LOCTN_OUT"
	else
		# one or none succeeded
		if [ -s "$SPOW_NETLC_OUT" ]; then
			if [ -n "$LOCATION_PRIORITISE_NETWORK" ]; then
				printf "using network (prioritised)" | tee -a "$SPOW_SESH_ERR"
			else
				printf "using network" | tee -a "$SPOW_SESH_ERR"
			fi
			cat "$SPOW_NETLC_OUT" >"$SPOW_LOCTN_OUT"
		fi
		if [ -s "$SPOW_GPSLC_OUT" ]; then
			printf "using gps" | tee -a "$SPOW_SESH_ERR"
			cat "$SPOW_GPSLC_OUT" >"$SPOW_LOCTN_OUT"
		fi
	fi
	if [ ! -s "$SPOW_LOCTN_OUT" ]; then
		printf "none (error)" | tee -a "$SPOW_SESH_ERR"
	fi
	printf "\n" | tee -a "$SPOW_SESH_ERR"
}

gensharetext() {
	surplus -td "$1" >"$SPOW_SPLUS_OUT" 2>"$SPOW_SPLUS_ERR"
	ret="$?"
	cat "$SPOW_SPLUS_OUT" >>"$SPOW_SESH_OUT"
	cat "$SPOW_SPLUS_ERR" >>"$SPOW_SESH_ERR"
	return "$ret"
}

send() {
	# $1 is sharetext

	# use fake if any
	fake_first=""
	fake_rest=""

	if [ -n "$(cat "$SPOW_FAKE_OUT")" ]; then
		inside_first_group=false
		first_group_done=false

		while IFS= read -r line; do
			# skip preceding empty lines
			if [ -z "$line" ] && [ "$inside_first_group" = false ] && [ -z "$fake_first" ]; then
				continue
			fi

			# if not empty and not inside first group, then we are inside first group
			if [ -n "$line" ] && [ "$inside_first_group" = false ] && [ "$first_group_done" = false ]; then
				inside_first_group=true

			# if empty and inside first group, then we are outside first group
			elif [ -z "$line" ] && [ "$inside_first_group" = true ]; then
				inside_first_group=false
				first_group_done=true

			# if empty and outside first group but first group is done, add newline to fake_rest
			elif [ -z "$line" ] && [ "$inside_first_group" = false ] && [ "$first_group_done" = true ]; then
				fake_rest="$fake_rest\n"
			fi

			# append to fake_first (if message_fake is not empty)
			if [ "$inside_first_group" = true ]; then
				if [ -z "$fake_first" ]; then
					fake_first="$line"
				else
					fake_first="$(printf "%s\n%s" "$fake_first" "$line")"
				fi

			# append to fake_rest (if message_fake is not empty)
			else
				if [ -z "$fake_rest" ]; then
					fake_rest="$line"
				else
					fake_rest="$(printf "%s\n%s" "$fake_rest" "$line")"
				fi
			fi

		done <"$SPOW_FAKE_OUT"

		if [ -n "$fake_first" ]; then
			printf "$fake_rest\n" >"$SPOW_FAKE_OUT"
		fi
	fi

	# choose what message to use
	message=""
	if [ -n "$fake_first" ]; then
		echo using fake
		message="$fake_first"
	else
		echo using message
		message="$1"
	fi

	# store message
	printf "%s\n" "$message" >"$SPOW_MESSAGE"
	cat "$SPOW_MESSAGE" >>"$SPOW_SESH_OUT"
	cat "$SPOW_MESSAGE" >>"$SPOW_SESH_ERR"

	# run bridges
	if [ -f "$SPOW_BRIDGES" ]; then
		# run commands in bridge file
		while IFS= read -r command; do
			# skip command if its actually a comment
			if [ "$(printf "%s" "$command" | head -c 1)" = "#" ]; then
				continue
			fi

			# run bridge
			echo "$command"
			echo "$SPOW_TARGETS" | eval "$command" >>"$SPOW_SESH_ERR" 2>>"$SPOW_SESH_ERR"
			bridge_return="$?"

			# check if bridge failed
			if [ ! "$bridge_return" -eq 0 ]; then
				bridge_failures=$((bridge_failures + 1))
			fi

			# store return value
			if [ -z "$bridge_returns" ]; then
				bridge_returns="$bridge_return"
			else
				bridge_returns="$bridge_returns, $bridge_return"
			fi
		done <"$SPOW_BRIDGES"
	else
		printf "s+ow: warning: no '$SPOW_BRIDGES' file; message is not sent.\n"
		termux-notification \
			--priority "default" \
			--title "surplus on wheels: No bridges" \
			--content "No '$SPOW_BRIDGES' file; message is not sent."
	fi

	echo "$bridge_returns" >>"$SPOW_SESH_ERR"
}

notify_start() {
	termux-notification \
		--priority "min" \
		--ongoing \
		--id "s+ow" \
		--title "surplus on wheels" \
		--content "s+ow has started running."
}

notify() {
	# $1 is text
	# $2 is attempt number (if any)
	attempt_text="$1"
	if [ $# -eq 2 ]; then
		attempt_text="$1 (attempt $2)"
	fi
	termux-notification \
		--priority "min" \
		--ongoing \
		--id "s+ow" \
		--title "surplus on wheels" \
		--content "$attempt_text"
}

notify_end() {
	# $1 is done tuple
	# $2 is sharetext
	termux-notification \
		--priority "min" \
		--id "s+ow" \
		--title "surplus on wheels" \
		--content "$(printf 'Run has finished. %s\n\n%s' "$1" "$2")"
}

# program functions

run() {
	notify_start
	printf "[run! stdout (%s)]\n" "$(date)" >>"$SPOW_SESH_OUT"
	printf "[run! stderr (%s)]\n" "$(date)" >>"$SPOW_SESH_ERR"

	time_run_start="$(date +%s)"

	# if cron: wait until its the new hour
	if [ -n "$SPOW_CRON" ]; then
		notify "Waiting for the 30th second to pass..."
		printf "waiting for the 30th second to pass...\n"
		while [ "$(date +'%S')" -lt 30 ]; do
			printf "  $(date)\n"
			sleep 1
		done
		printf "proceeding\n"
	fi

	time_locate_start="$(date +%s)"

	# termux-location
	location=""
	for locate_run in 1 2 3; do # run three times in case :p
		notify "Running termux-location" "$locate_run"

		if [ "$locate_run" -gt "1" ]; then
			LOCATION_TIMEOUT=75 locate
		else
			locate
		fi
		if [ ! -s "$SPOW_LOCTN_OUT" ]; then
			# erroneous: is empty
			echo "s+ow: error: failed to get location" | tee -a "$SPOW_SESH_ERR"
			status=1
		else
			# nominal: is not empty
			location="$(cat "$SPOW_LOCTN_OUT")"
			status=0
			break
		fi
	done

	time_locate_end="$(date +%s)"
	time_locate=$((time_locate_end - time_locate_start))

	time_surplus_start="$(date +%s)"

	# surplus
	printf "running surplus... "
	notify "Running surplus -td $location"
	if [ "$status" -eq 0 ]; then
		if gensharetext "$location"; then
			# surplus ran nominally
			cp "$SPOW_SPLUS_OUT" "$SPOW_LAST_OUT"
			status=0
			printf "nominal\n"
		else
			# something happened :^)
			status=2
			printf "errored\n"
		fi
	else
		printf "skipped\n"
	fi

	time_surplus_end="$(date +%s)"
	time_surplus=$((time_surplus_end - time_surplus_start))

	# if cron: wait until its the new hour
	if [ -n "$SPOW_CRON" ]; then
		notify "Waiting for the new hour..."
		printf "waiting until the new hour...\n"
		while [ "$(date +'%M')" -eq 59 ]; do
			printf "  $(date)\n"
			sleep 1
		done
		printf "proceeding\n"
	fi

	time_sendmsg_start="$(date +%s)"

	# mdtest/send message
	printf "sending message(s)... "
	notify "Sending message(s)"
	sent_type=0 # 0 for freshly made sharetext
	# 1 for recycling a last location
	# 2 for using fallback template
	bridge_failures=0
	sharetext=""
	send_error_notif=false
	if [ "$status" -eq 0 ]; then
		# s+ow has behaved nominally until now, send as per normal
		sharetext="$(cat "$SPOW_SPLUS_OUT")"
		printf "\n"

		send "$sharetext"
	else
		# something has gone wrong, send an appropriate fallback
		sharetext=""
		if [ -s "$SPOW_LAST_OUT" ]; then
			# use last successful location
			sharetext="$(cat "$SPOW_LAST_OUT")"
			sent_type=1
			printf "using last...\n"

		else
			# no last location, use fallback
			# shellcheck disable=SC2059
			sharetext="$(printf "$LOCATION_FALLBACK" "$status" "$locate_run" "$sent_type")"
			sent_type=2
			printf "using fallback... \n"

		fi

		send "$sharetext"

		# send done info except
		printf "(%d, %d, %d, %s<%s>) [lc:%ds sp:%ds]\n" "$status" "$locate_run" "$sent_type" "$bridge_failures" "$bridge_returns" "$time_locate" "$time_surplus" >>"$SPOW_SESH_ERR"

		send_error_notif=true
	fi

	time_sendmsg_end="$(date +%s)"
	time_sendmsg=$((time_sendmsg_end - time_sendmsg_start))

	time_run_end="$(date +%s)"
	time_run=$((time_run_end - time_run_start))

	done_info="$(printf "(%d, %d, %d, %s<%s>) [lc:%ds sp:%ds sm:%ds - %ds]\n" "$status" "$locate_run" "$sent_type" "$bridge_failures" "$bridge_returns" "$time_locate" "$time_surplus" "$time_sendmsg" "$time_run")"
	echo "done $done_info" | tee -a "$SPOW_SESH_ERR"

	# finish
	notify_end "$done_info" "$sharetext"
	if [ ! "$send_error_notif" = false ]; then
		# error notification
		termux-notification \
			--priority "default" \
			--title "surplus on wheels has errored" \
			--content "$(printf "\n(%d, %d, %d, %s<%s>)\n[lc:%ds sp:%ds sm:%ds - %ds]\n" "$status" "$locate_run" "$sent_type" "$bridge_failures" "$bridge_returns" "$time_locate" "$time_surplus" "$time_sendmsg" "$time_run")"
	fi
}

# script entry

if [ -z "$SPOW_TARGETS" ]; then
	echo "s+ow: error: SPOW_TARGETS are not set"
	exit 1
fi
run

printf "%s\n\n" "$(cat "$SPOW_SESH_OUT")" >>"$SPOW_WEEK_OUT"
printf "%s\n\n" "$(cat "$SPOW_SESH_ERR")" >>"$SPOW_WEEK_ERR"
