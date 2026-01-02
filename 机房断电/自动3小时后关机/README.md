hosts=(UM246 UM247 UM248 UM249 UM250 UM251 UM252 UM253)

printf "sudo password: "
stty -echo; read SUDOPASS; stty echo; printf "\n"

for h in "${hosts[@]}"; do
  echo "==> $h"
  printf '%s\n' "$SUDOPASS" | ssh -o ConnectTimeout=5 "$h" \
    "sudo -S shutdown -h +300 'Scheduled poweroff (batch 1)'" \
    && echo "OK $h" || echo "FAIL $h"
done

echo "==> UM139"
printf '%s\n' "$SUDOPASS" | ssh UM139 "sudo -S shutdown -h +310 'Scheduled poweroff (after batch 1)'" \
  && echo "OK UM139" || echo "FAIL UM139"

unset SUDOPASS
