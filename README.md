
# Twitch VOD to YouTube Uploader

This script automates the process of uploading Twitch VODs recorded via OBS to YouTube. It retrieves metadata from Twitch (like the game directory) and associates it with your video uploads, ensuring seamless and organized video management.

---

## Features
- **Fetch Latest Twitch VODs**: Automatically retrieves your recent VODs.
- **Game Metadata Integration**: Adds the game title as a YouTube tag.
- **Automated Upload**: Matches local OBS recordings with Twitch VODs and uploads them to YouTube.
- **Duplicate Detection**: Checks if a Twitch VOD has already been uploaded to YouTube by searching video descriptions.
- **Resumable OAuth2 Authentication**: No need for repeated manual logins after the first run.
- **Task Scheduling (Optional)**: Automate periodic execution using a built-in task scheduler configuration.

---

## How It Works

1. **Checking for Existing Videos**:
   - The script searches your YouTube channel for videos with descriptions containing the **Twitch VOD ID**.
   - If a video matching the VOD ID is found, the VOD is considered already uploaded and skipped.

2. **Matching Twitch VODs with Local Recordings**:
   - Each Twitch VOD's creation time is retrieved and compared with local OBS recording timestamps (within a 1-hour tolerance).
   - If a match is found, the script proceeds to upload the video.

3. **Uploading the VOD**:
   - If no matching video is found on YouTube, and a local recording is available, the script uploads the recording.
   - The upload includes metadata like the Twitch VOD ID, title, and game name, which is added as a YouTube tag.

4. **Task Scheduling (Optional)**:
   - The script can create a scheduled task to run periodically using **Windows Task Scheduler**.
   - You can enable this functionality by setting `schedule_task: true` in the `config.json` file.
   - Specify the desired frequency (`daily`, `weekly`, or `hourly`) in the configuration.

---

## Setup

### Prerequisites
1. **Python 3.7 or higher**.
2. **Python Packages**:
   Install the required packages using pip:
   ```bash
   pip install google-api-python-client google-auth google-auth-oauthlib requests pytz
   ```
3. **OBS Studio with Auto Recording Enabled**:
   - Open OBS.
   - Go to **Settings** > **Output**.
   - Ensure the **Recording Path** is set to a folder you can access easily (this will be your `recordings_dir` in `config.json`).
   - Under the **General** tab, enable **Automatically record when streaming**. This ensures OBS will create a local recording of every stream.

---

### Configuration

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-username/twitch-vod-uploader.git
   cd twitch-vod-uploader
   ```

2. **Prepare Your `config.json`**:
   Create a `config.json` file in the project directory with the following content:
   ```json
   {
     "twitch_client_id": "your_twitch_client_id",
     "twitch_client_secret": "your_twitch_client_secret",
     "twitch_user_id": "your_twitch_user_id",
     "youtube_client_secrets_file": "client_secret.json",
     "youtube_channel_id": "your_channel_id",
     "recordings_dir": "C:/path/to/your/recordings",
     "schedule_task": true,
     "task_frequency": "daily"  // Options: "daily", "weekly", "hourly"
   }
   ```
   - **Replace**:
     - `"your_twitch_client_id"` with your Twitch Client ID.
     - `"your_twitch_client_secret"` with your Twitch Client Secret.
     - `"your_twitch_user_id"` with your Twitch User ID.
     - `"your_channel_id"` with your YouTube Channel ID.
     - `"C:/path/to/your/recordings"` with the path to your OBS recordings directory.
   - **Optional**:
     - Set `"schedule_task"` to `true` to enable automatic task scheduling.
     - Set `"task_frequency"` to your desired frequency (`"daily"`, `"weekly"`, or `"hourly"`).

3. **Get Your YouTube Client Secrets**:
   - Go to the [Google Cloud Console](https://console.cloud.google.com/).
   - **Create a Project** and enable the **YouTube Data API v3**.
   - **Generate OAuth 2.0 Credentials**:
     - Navigate to **APIs & Services** > **Credentials**.
     - Click **Create Credentials** > **OAuth client ID**.
     - Select **Desktop app** and proceed.
     - Download the `client_secret.json` file and place it in the project directory.

4. **Run the Script**:
   ```bash
   python TwitchVodUploader.py
   ```
   - **First Run**: The script will prompt you to authenticate via OAuth in your browser. This will create a `token.json` file in the script’s directory for future authentication.
   - **Subsequent Runs**: The script will use the stored `token.json` to authenticate automatically without manual intervention.

5. **Task Scheduling (Optional)**:
   - If `schedule_task` is set to `true` in the `config.json`, the script will automatically create a task in **Windows Task Scheduler** to run periodically based on `task_frequency`.
   - **Example Frequencies**:
     - `"daily"`: Runs every day at the specified time.
     - `"weekly"`: Runs every week at the specified time.
     - `"hourly"`: Runs every hour.

   **Note**: Ensure the script has the necessary permissions to create scheduled tasks on your system.

---

## License
© 2024 Tim Weber

All rights reserved.

Permission to use, copy, modify, and/or distribute this software for any purpose is granted only to individuals or entities who have obtained explicit written permission from the copyright owner.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES, OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT, OR OTHERWISE, ARISING FROM, OUT OF, OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

For inquiries about licensing this software, please contact:
Tim Weber
timweberj@gmail.com


## Contact
For any questions or permissions regarding this script, please contact:
- **Email**: timweberj@gmail.com
- **GitHub**: [TeeJay69](https://github.com/TeeJay69)

---

### Additional Notes:

- **Scheduled Task Functionality Through Python**:
  - The script includes the ability to create a scheduled task using Python’s `subprocess` module and Windows’ `schtasks.exe`.
  - When `schedule_task` is enabled in `config.json`, the script will create a task named `TwitchVODUploader` with the specified frequency.
  - Users can disable this by setting `schedule_task` to `false` in the configuration.

- **Ensuring Proper Timezone Configuration**:
  - The script assumes your local timezone is `"Europe/Berlin"`. If you are in a different timezone, update the `find_matching_recording` function accordingly.

- **Example `config_template.json`**:
  - It’s a good practice to include a template configuration file in your repository. Users can copy this and fill in their own details.
  
```json
  {
    "twitch_client_id": "your_twitch_client_id",
    "twitch_client_secret": "your_twitch_client_secret",
    "twitch_user_id": "your_twitch_user_id",
    "youtube_client_secrets_file": "client_secret.json",
    "youtube_channel_id": "your_channel_id",
    "recordings_dir": "C:/path/to/your/recordings",
    "schedule_task": true,
    "_comment_task_frequency": "Options: daily, weekly, hourly",
    "task_frequency": "daily"
```

---