# AGENTS.md

## Deploy to PhotonVision device (`pi@photonvision.local`)

Use this workflow when deploying a locally built jar to a PhotonVision device.

### Goal
- Build the Linux ARM64 PhotonVision jar locally.
- Copy it to `/opt/photonvision/` on the target.
- Back up the existing `/opt/photonvision/photonvision.jar`.
- Update `/opt/photonvision/photonvision.jar` to a symlink pointing at the new jar.

### Prerequisites
- You can SSH to `pi@photonvision.local`.
- If local SSH keys do not work, use password `raspberry`.
- `sudo` on the target user.

### 1) Build locally
From repo root:

```bash
BUILD_START=$(date +%s)

# Build native targeting artifacts first (avoids missing wpilibNatives zips in some environments)
./gradlew :photon-targeting:build

# Then build the deployable ARM64 server jar.
# Plain `:photon-server:shadowJar` builds the host-platform jar on this machine,
# typically at `photon-server/build/libs/photonvision-*-linuxx64.jar`.
# The ARM64 deploy jar is written to
# `photon-server/build/libs/photonvision-*-linuxarm64.jar`.
./gradlew :photon-server:shadowJar -PArchOverride=linuxarm64 -Ponlylinuxarm64
```

Find the newest built ARM64 jar:

```bash
NEW_JAR=$(ls -1t photon-server/build/libs/photonvision*-linuxarm64.jar | head -n 1)
echo "$NEW_JAR"

# Ensure it was built after this deploy session started
NEW_JAR_MTIME=$(stat -c %Y "$NEW_JAR")
if [ "$NEW_JAR_MTIME" -lt "$BUILD_START" ]; then
  echo "No new linuxarm64 jar produced by this build; aborting deploy."
  exit 1
fi
```

### 2) Copy jar to target
Preferred (SSH keys):

```bash
scp "$NEW_JAR" pi@photonvision.local:/tmp/
```

Password fallback (`raspberry`) if keys fail:

```bash
sshpass -p raspberry scp "$NEW_JAR" pi@photonvision.local:/tmp/
```

### 3) Backup current jar and switch symlink
Run on target (keys):

```bash
ssh pi@photonvision.local 'set -euo pipefail
cd /opt/photonvision
NEW_REMOTE_JAR="/tmp/'"$(basename "$NEW_JAR")"'"
DEST_JAR="/opt/photonvision/'"$(basename "$NEW_JAR")"'"
TS=$(date +%Y%m%d-%H%M%S)

sudo systemctl stop photonvision.service

if [ -e /opt/photonvision/photonvision.jar ] || [ -L /opt/photonvision/photonvision.jar ]; then
  OLD_TARGET=$(readlink -f /opt/photonvision/photonvision.jar || true)
  sudo cp -a /opt/photonvision/photonvision.jar "/opt/photonvision/photonvision.jar.bak-$TS"
  if [ -n "$OLD_TARGET" ] && [ -f "$OLD_TARGET" ]; then
    sudo cp -a "$OLD_TARGET" "/opt/photonvision/$(basename "$OLD_TARGET").bak-$TS"
  fi
fi

sudo mv "$NEW_REMOTE_JAR" "$DEST_JAR"
sudo ln -sfn "$DEST_JAR" /opt/photonvision/photonvision.jar
sudo chmod +x "$DEST_JAR"

sudo systemctl start photonvision.service
sudo systemctl status photonvision.service --no-pager -l | sed -n "1,25p"
ls -l /opt/photonvision/photonvision.jar
'
```

Password fallback (`raspberry`) version:

```bash
sshpass -p raspberry ssh pi@photonvision.local 'set -euo pipefail
cd /opt/photonvision
NEW_REMOTE_JAR="/tmp/'"$(basename "$NEW_JAR")"'"
DEST_JAR="/opt/photonvision/'"$(basename "$NEW_JAR")"'"
TS=$(date +%Y%m%d-%H%M%S)

sudo systemctl stop photonvision.service

if [ -e /opt/photonvision/photonvision.jar ] || [ -L /opt/photonvision/photonvision.jar ]; then
  OLD_TARGET=$(readlink -f /opt/photonvision/photonvision.jar || true)
  sudo cp -a /opt/photonvision/photonvision.jar "/opt/photonvision/photonvision.jar.bak-$TS"
  if [ -n "$OLD_TARGET" ] && [ -f "$OLD_TARGET" ]; then
    sudo cp -a "$OLD_TARGET" "/opt/photonvision/$(basename "$OLD_TARGET").bak-$TS"
  fi
fi

sudo mv "$NEW_REMOTE_JAR" "$DEST_JAR"
sudo ln -sfn "$DEST_JAR" /opt/photonvision/photonvision.jar
sudo chmod +x "$DEST_JAR"

sudo systemctl start photonvision.service
sudo systemctl status photonvision.service --no-pager -l | sed -n "1,25p"
ls -l /opt/photonvision/photonvision.jar
'
```

### 4) Quick verification

```bash
# Should show photonvision.jar -> /opt/photonvision/photonvision-...-linuxarm64.jar
ssh pi@photonvision.local 'ls -l /opt/photonvision/photonvision.jar'

# Recent service logs
ssh pi@photonvision.local 'sudo journalctl -u photonvision.service -n 80 --no-pager -l -o cat'
```

If keys are unavailable, prepend `sshpass -p raspberry` to the `ssh` commands above.
