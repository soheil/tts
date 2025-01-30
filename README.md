# tts
Instantly listen to highlighted text on MacOS. It works by creating an Apple service and assigning a hotkey to it.

<audio controls>
  <source src="./sample.mp4" type="audio/mpeg">
  Your browser does not support the audio tag.
</audio>

This works by creating an `Apple Automator` service in your `~/Library/Services` to run a shell script and call OpenAI text-to-speech API (which works incredibly well and sounds very natural.)

## Installation

### Option A (simple)

1. Add your `OPENAI_API_KEY` in: [tts.workflow/Contents/document.wflow](./tts.workflow/Contents/document.wflow)
2. `cp -r tts.workflow ~/Library/Services/`
3. Add a keyboard hotkey (I use `CMD+Shift+.`) by opening: System Settings > Keyboard > Keyboard Shortcuts (button) > Services (tab) > Text (dropdown) -> [assign any hotkey to an item called `tts`]
4. Select any text in any app and press your hotkey to listen to it.


### (Option B)

1. Open Automator app on your Mac
2. Go to File menu and click on New then select Quick Action as type
3. Once opened scroll down to "Run Shell Script" inside Library and double click to add it
4. Copy and paste script below inside it (make sure to add your `OPENAI_API_KEY`)
5. Add a keyboard hotkey (I use `CMD+Shift+.`) by opening: System Settings > Keyboard > Keyboard Shortcuts (button) > Services (tab) > Text (dropdown) -> [assign any hotkey to an item called `tts`]
6. Select any text in any app and press your hotkey to listen to it.

```bash
#!/bin/bash

# Read and clean up input text
TEXT=$(cat /dev/stdin)
TEXT=$(echo "$TEXT" | tr '\n' ' ')

# Set your OpenAI API Key
OPENAI_API_KEY=

# Maximum characters for the first segment for quick startup
FIRST_SEGMENT_CHARS=120

# Approximate maximum number of characters per API call
MAX_CHARS=1000

# Directory for temporary audio files
TMP_DIR=$(mktemp -d)

# Function to make an API call, save audio, and notify completion
api_call() {
  local text_segment=$1
  local file_output=$2
  
  json_payload=$(cat <<EOF
{
  "model": "tts-1",
  "input": "$text_segment",
  "voice": "nova"
}
EOF
)

  curl -s https://api.openai.com/v1/audio/speech \
      -H "Authorization: Bearer $OPENAI_API_KEY" \
      -H "Content-Type: application/json" \
      -d "$json_payload" \
      --output "$file_output"
}

# --- Begin Split Operations ---

# Handle the first segment specifically to ensure it's short
first_segment=$(echo "$TEXT" | cut -c1-$FIRST_SEGMENT_CHARS)
remaining_text=$(echo "$TEXT" | cut -c$((FIRST_SEGMENT_CHARS + 1))-)

# Split the remaining text into manageable chunks
split_texts=("$first_segment")
while read -r line; do
  split_texts+=("$line")
done &lt; &lt;(echo "$remaining_text" | /usr/bin/fold -s -w $MAX_CHARS)

# --- End Split Operations ---

# Create a temporary file to hold our PIDs
PID_FILE=$(mktemp)

# Start API calls in parallel &amp; write PIDs
for i in "${!split_texts[@]}"; do
  api_call "${split_texts[$i]}" "$TMP_DIR/speech_$i.mp3" &amp;
  echo $! &gt;&gt; "$PID_FILE"
done

# Play audio files in order of their completion
for i in "${!split_texts[@]}"; do
  wait $(sed -n "$((i + 1))p" "$PID_FILE")
  afplay "$TMP_DIR/speech_$i.mp3"
done

# Clean up
rm -r "$TMP_DIR"
```


The script is optimized to play the first chunk of text right away so you don't have to wait for a long API call.
