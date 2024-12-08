import os
import json
import subprocess
from datetime import datetime, timedelta
import requests
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
import pytz  # Ensure pytz is installed

# Get the current script's directory
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))

# Paths for token and cache files
TOKEN_FILE = os.path.join(SCRIPT_DIR, "token.json")
CACHE_FILE = os.path.join(SCRIPT_DIR, "uploaded_videos_cache.json")
CONFIG_FILE = os.path.join(SCRIPT_DIR, "config.json")

# Read configuration from `config.json`
with open(CONFIG_FILE, "r") as f:
    config = json.load(f)

# Twitch API credentials
TWITCH_CLIENT_ID = config["twitch_client_id"]
TWITCH_CLIENT_SECRET = config["twitch_client_secret"]

# YouTube API credentials
CLIENT_SECRETS_FILE = os.path.join(SCRIPT_DIR, config["youtube_client_secrets_file"])
CHANNEL_ID = config["youtube_channel_id"]

# Local recordings directory
RECORDINGS_DIR = config["recordings_dir"]

# YouTube API Scopes
SCOPES = ["https://www.googleapis.com/auth/youtube.upload"]


def get_authenticated_service():
    """Authenticate with the YouTube API using OAuth 2.0."""
    creds = None
    if os.path.exists(TOKEN_FILE):
        creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)

    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(CLIENT_SECRETS_FILE, SCOPES)
            creds = flow.run_local_server(port=8080)
            with open(TOKEN_FILE, "w") as token:
                token.write(creds.to_json())

    return build("youtube", "v3", credentials=creds)


def get_twitch_access_token():
    """Get Twitch access token using the client credentials."""
    url = "https://id.twitch.tv/oauth2/token"
    payload = {
        "client_id": TWITCH_CLIENT_ID,
        "client_secret": TWITCH_CLIENT_SECRET,
        "grant_type": "client_credentials"
    }
    response = requests.post(url, data=payload)
    response.raise_for_status()
    return response.json()["access_token"]


TWITCH_ACCESS_TOKEN = get_twitch_access_token()


def get_latest_twitch_vods():
    """Fetch the latest VODs from the Twitch API."""
    url = "https://api.twitch.tv/helix/videos"
    headers = {
        "Client-ID": TWITCH_CLIENT_ID,
        "Authorization": f"Bearer {TWITCH_ACCESS_TOKEN}"
    }
    params = {"user_id": config["twitch_user_id"], "type": "archive"}
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    vods = response.json()["data"]

    for vod in vods:
        game_id = vod.get("game_id")
        vod["game_name"] = get_game_name(game_id) if game_id else "Unknown Game"

    return vods


def get_game_name(game_id):
    """Fetch the name of the game associated with a game ID."""
    url = "https://api.twitch.tv/helix/games"
    headers = {
        "Client-ID": TWITCH_CLIENT_ID,
        "Authorization": f"Bearer {TWITCH_ACCESS_TOKEN}"
    }
    params = {"id": game_id}
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    data = response.json().get("data", [])
    return data[0].get("name", "Unknown Game") if data else "Unknown Game"


def load_cache():
    """Load the cache of uploaded videos."""
    if os.path.exists(CACHE_FILE):
        with open(CACHE_FILE, "r") as f:
            return json.load(f)
    return []


def save_cache(data):
    """Save the cache of uploaded videos."""
    with open(CACHE_FILE, "w") as f:
        json.dump(data, f, indent=2)


def update_cache_with_new_video(vod_id):
    """Add a newly uploaded video to the cache."""
    cache = load_cache()
    cache.append(vod_id)
    save_cache(cache)


def find_matching_recording(vod):
    """Find a local recording that matches the Twitch VOD creation time."""
    vod_time = datetime.strptime(vod["created_at"], "%Y-%m-%dT%H:%M:%SZ").replace(tzinfo=pytz.utc)

    for file in os.listdir(RECORDINGS_DIR):
        if not file.endswith(".mp4"):
            continue

        try:
            file_time = datetime.strptime(file[:19], "%Y-%m-%d %H-%M-%S")
            file_time = file_time.replace(tzinfo=pytz.timezone("Europe/Berlin"))
            file_time_utc = file_time.astimezone(pytz.utc)
        except ValueError:
            continue

        if abs((vod_time - file_time_utc).total_seconds()) < 3600:
            return os.path.join(RECORDINGS_DIR, file)

    return None


def upload_to_youtube(file_path, title, vod_id, game_name):
    """Upload a video to YouTube."""
    youtube = get_authenticated_service()
    media = MediaFileUpload(file_path, chunksize=-1, resumable=True)
    request = youtube.videos().insert(
        part="snippet,status",
        body={
            "snippet": {
                "title": title,
                "description": f"Twitch VOD ID: {vod_id}\nAutomatically uploaded.",
                "tags": ["twitch", "stream", "vod", game_name]
            },
            "status": {
                "privacyStatus": "public",
                "selfDeclaredMadeForKids": False
            }
        },
        media_body=media
    )
    return request.execute()


def create_scheduled_task(frequency):
    """Create a Windows Task Scheduler task to run this script periodically."""
    task_name = "TwitchVODUploader"
    script_path = os.path.abspath(__file__)
    command = f"schtasks /create /TN {task_name} /TR \"python {script_path}\" /F"

    if frequency == "daily":
        command += " /SC DAILY /ST 09:00"
    elif frequency == "weekly":
        command += " /SC WEEKLY /D MON /ST 09:00"
    elif frequency == "hourly":
        command += " /SC HOURLY"
    else:
        print("Invalid frequency specified. Options: daily, weekly, hourly.")
        return

    # Check if the task already exists
    try:
        existing_task = subprocess.run(
            ["schtasks", "/Query", "/TN", task_name],
            capture_output=True,
            text=True
        )
        if existing_task.returncode == 0:
            print(f"Scheduled task '{task_name}' already exists. Skipping creation.")
            return
    except Exception as e:
        print(f"Error checking existing task: {e}")
        return

    # Create the task if it does not already exist
    try:
        subprocess.run(command, shell=True, check=True)
        print(f"Scheduled task created: {task_name}")
    except subprocess.CalledProcessError as e:
        print(f"Failed to create scheduled task: {e}")



def main():
    """Main entry point for the script."""
    # Task scheduling logic
    if config.get("schedule_task", False):
        create_scheduled_task(config.get("task_frequency", "daily"))

    # Proceed with the main logic even if the task was just scheduled
    vods = get_latest_twitch_vods()
    cache = load_cache()

    for vod in vods:
        vod_id = vod["id"]
        game_name = vod["game_name"]

        if vod_id in cache:
            print(f"VOD {vod_id} is already uploaded. Skipping.")
            continue

        file_path = find_matching_recording(vod)
        if not file_path:
            print(f"No matching recording found for VOD {vod_id}.")
            continue

        print(f"Uploading VOD {vod_id} to YouTube...")
        upload_response = upload_to_youtube(file_path, vod["title"], vod_id, game_name)
        print(f"Uploaded video with YouTube ID: {upload_response['id']}")
        update_cache_with_new_video(vod_id)

if __name__ == "__main__":
    main()
