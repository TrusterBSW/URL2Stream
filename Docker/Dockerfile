FROM debian:bookworm-slim

# Install dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
    xvfb \
    firefox-esr \
    ffmpeg \
    xdotool \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Environment variables with defaults
ENV DISPLAY=:99
ENV XVFB_WIDTH=1920
ENV XVFB_HEIGHT=1080
ENV XVFB_DEPTH=24

ENV FFMPEG_FRAMERATE=24
ENV FFMPEG_CRF=26
ENV FFMPEG_CODEC=libx264
ENV FFMPEG_PRESET=ultrafast
ENV FFMPEG_CONTAINER=mpegts
ENV FFMPEG_OUTPUT_URI=udp://172.17.0.1:9000

ENV FIREFOX_URL=https://example.com

# Create entrypoint script
RUN cat > /app/entrypoint.sh << 'EOF'
#!/bin/bash
set -e

echo "[*] Starting Firefox to Video Stream Pipeline"
echo "[*] Configuration:"
echo "    Display: $DISPLAY"
echo "    Resolution: ${XVFB_WIDTH}x${XVFB_HEIGHT}x${XVFB_DEPTH}"
echo "    Framerate: $FFMPEG_FRAMERATE fps"
echo "    CRF: $FFMPEG_CRF"
echo "    Codec: $FFMPEG_CODEC"
echo "    Container: $FFMPEG_CONTAINER"
echo "    Output: $FFMPEG_OUTPUT_URI"
echo "    Firefox URL: $FIREFOX_URL"


# Start Xvfb
echo "[*] Starting Xvfb on $DISPLAY with resolution ${XVFB_WIDTH}x${XVFB_HEIGHT}x${XVFB_DEPTH}..."
Xvfb $DISPLAY -screen 0 ${XVFB_WIDTH}x${XVFB_HEIGHT}x${XVFB_DEPTH} -ac -noreset -nocursor &
XVFB_PID=$!
sleep 2

# Verify Xvfb is running
if ! kill -0 $XVFB_PID 2>/dev/null; then
    echo -e "\t[!] ERROR: Xvfb failed to start"
    exit 1
fi
echo -e "\t[+] Xvfb started (PID: $XVFB_PID)"

# Start ffmpeg streaming
echo "[*] Starting ffmpeg stream to $FFMPEG_OUTPUT_URI..."
ffmpeg -nostdin -f x11grab \
    -r $FFMPEG_FRAMERATE \
    -s ${XVFB_WIDTH}x${XVFB_HEIGHT} \
    -i $DISPLAY.0 \
    -c:v $FFMPEG_CODEC \
    -preset $FFMPEG_PRESET \
    -crf $FFMPEG_CRF \
    -f $FFMPEG_CONTAINER \
    "$FFMPEG_OUTPUT_URI" \
    > /dev/null 2>&1 &

FFMPEG_PID=$!
sleep 2

# Verify ffmpeg is running
if ! kill -0 $FFMPEG_PID 2>/dev/null; then
    echo -e "\t[!] ERROR: ffmpeg failed to start"
    kill $FIREFOX_PID $XVFB_PID
    exit 1
fi
echo -e "\t[+] ffmpeg streaming started (PID: $FFMPEG_PID)"


# Start Firefox
FIREFOX_CMD="firefox --no-remote --kiosk"
if [ -d "$FIREFOX_PROFILE" ]; then
    FIREFOX_CMD="$FIREFOX_CMD --profile $FIREFOX_PROFILE"
    echo "[*] Using Firefox profile: $FIREFOX_PROFILE"
else
    echo "[*] Firefox profile not found or not defined, using default"
fi
echo "[*] Starting Firefox with URL: $FIREFOX_URL"
FIREFOX_CMD="$FIREFOX_CMD $FIREFOX_URL"
eval "$FIREFOX_CMD" > /dev/null 2>&1 &
FIREFOX_PID=$!
sleep 4

# Verify Firefox is running
if ! kill -0 $FIREFOX_PID 2>/dev/null; then
    echo -e "\t[!] ERROR: Firefox failed to start"
    kill $XVFB_PID
    exit 1
fi
echo -e "\t[+] Firefox started (PID: $FIREFOX_PID)"

# Resize Firefox window to match Xvfb resolution
echo "[*] Resizing Firefox window to ${XVFB_WIDTH}x${XVFB_HEIGHT}..."
sleep 1
xdotool search --sync --onlyvisible --name "Mozilla Firefox" windowsize $XVFB_WIDTH $XVFB_HEIGHT
sleep 1
xdotool mousemove $XVFB_WIDTH $XVFB_HEIGHT
echo -e "\t[+] Firefox window resized"

echo "[*] Pipeline running. Stream available at: $FFMPEG_OUTPUT_URI"

# Custom Script to control Browser
if [ -f "$CUSTOM_SCRIPT" ]; then
    echo "[*] Running custom script $CUSTOM_SCRIPT"
    bash $CUSTOM_SCRIPT
fi

# Setup cleanup on exit
cleanup() {
    echo "[*] Shutting down..."
    kill $FFMPEG_PID 2>/dev/null || true
    sleep 1
    kill $FIREFOX_PID 2>/dev/null || true
    kill $XVFB_PID 2>/dev/null || true
    echo "[+] Cleanup complete"
}

trap cleanup EXIT INT TERM

# Wait indefinitely
wait $FFMPEG_PID
EOF

RUN ["chmod", "+x", "/app/entrypoint.sh"]

ENTRYPOINT ["/app/entrypoint.sh"]
