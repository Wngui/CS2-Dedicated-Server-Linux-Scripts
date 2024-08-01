# CS2 Dedicated Server Linux Scripts

I collection of scripts, useful for managing and updating a CS2 dedicated server.

Has support to automatically update CS2, Metamod and CounterStrikeSharp

## Setup

Add the following to your .bashrc file
Change paths where applicable, **/home/steam/steamcmd/cs2-ds/**

```bash
cs2() {
    case "$1" in
        start)
            sudo chown -R steam:steam /home/steam/steamcmd/ #Ensure folders and files are accessible
            if pgrep -u steam -f '/home/steam/steamcmd/cs2-ds/game/bin/linuxsteamrt64/cs2' > /dev/null; then
                echo "CS2 dedicated server is already running."
            else
                sudo -u steam screen -dmS cs2-ds /home/steam/steamcmd/cs2-ds/game/bin/linuxsteamrt64/cs2 -dedicated -maxplayers 60 -tickrate 128 +game_mode 0 +game_type 3 +map de_dust2 +sv_setsteamaccount DB3AD3DD1C02F7DB8E18F5B1F805ECBB
                echo "CS2 dedicated server started."
            fi
            ;;
        stop)
            pkill -u steam -f '/home/steam/steamcmd/cs2-ds/game/bin/linuxsteamrt64/cs2'
            echo "CS2 dedicated server stopped."
            ;;
        restart)
            cs2 stop && cs2 start
            ;;
        console)
            sudo -u steam screen -d -r cs2-ds
            ;;
        update)
            cs2 stop
            sudo -u steam cp /home/steam/steamcmd/cs2-ds/game/csgo/cfg/server.cfg -f /home/steam/steamcmd/cs2-ds/game/csgo/cfg/server_backup.cfg
            sudo -u steam /home/steam/steamcmd.sh +login anonymous +force_install_dir /home/steam/steamcmd/cs2-ds +app_update 730 validate +quit
            sudo -u steam cp /home/steam/steamcmd/cs2-ds/game/csgo/cfg/server_backup.cfg -f /home/steam/steamcmd/cs2-ds/game/csgo/cfg/server.cfg
            cs2 metamod
            cs2 csharp
            cs2 start
            ;;
        metamod)
            echo "Updating metamod..."
            latestDownload=$(curl -s "https://mms.alliedmods.net/mmsdrop/2.0/mmsource-latest-linux")
            sudo -u steam curl "https://mms.alliedmods.net/mmsdrop/2.0/$latestDownload" --create-dirs -o "/home/steam/downloads/$latestDownload"
            echo "Installing metamod version $latestDownload"
            sudo -u steam tar -xzf "/home/steam/downloads/$latestDownload" -C "/home/steam/steamcmd/cs2-ds/game/csgo/" --overwrite
            #clean up
            rm -f "/home/steam/downloads/$latestDownload"

            #Remove gameinfo entries
            sudo -u steam sed -i '/^\t\t\tGame\tcsgo\/addons\/metamod/d' /home/steam/steamcmd/cs2-ds/game/csgo/gameinfo.gi

            #Readd gameinfo entries
            sudo -u steam sed -i '/Game\tcsgo\s/i \\t\t\tGame\tcsgo/addons/metamod' /home/steam/steamcmd/cs2-ds/game/csgo/gameinfo.gi
            ;;
        csharp)
            echo "Updating counterstrike-sharp..."
            latestDownload=$(curl -s "https://api.github.com/repos/roflmuffin/CounterStrikeSharp/releases/latest" | grep -oP 'https:\/\/github.com\/roflmuffin\/CounterStrikeSharp\/releases\/download\/v\d+\/counterstrikesharp-with-runtime-build-\d+-linux-[a-f0-9]+\.zip') 
            zipFile=$(basename "$latestDownload")
            sudo -u steam curl "$latestDownload" --create-dirs -L -o "/home/steam/downloads/$zipFile"
            echo "Installing counterstrike-sharp version $zipFile"
            sudo -u steam unzip -o "/home/steam/downloads/$zipFile" -d "/home/steam/steamcmd/cs2-ds/game/csgo/"
            #clean up
            rm -f "/home/steam/downloads/$zipFile"
            ;;
        status)
            if pgrep -u steam -f '/home/steam/steamcmd/cs2-ds/game/bin/linuxsteamrt64/cs2' > /dev/null; then
                echo "CS2 dedicated server is running."
            else
                echo "CS2 dedicated server not running."
            fi
            ;;
        checkupdate)
            current_patch_version=$(grep "^PatchVersion=" "/home/steam/steamcmd/cs2-ds/game/csgo/steam.inf" | cut -d'=' -f2 | tr -d '[:space:].')
            response=$(curl -s "https://api.steampowered.com/ISteamApps/UpToDateCheck/v1/?appid=730&version=$current_patch_version")
            up_to_date=$(echo $response | jq -r '.response.up_to_date')

            if [ $up_to_date != true ]; then
                cs2 update
            fi
            ;;
        *)
            echo "Usage: cs2 {start|stop|restart|console|update|status}"
            ;;
    esac
}
```

## Usage
```cs2 start``` - Starts the CS2 dedicated server if not already running
```cs2 stop``` - Stops the CS2 dedicated server if running
```cs2 restart``` - Stops and then starts the CS2 dedicated server
```cs2 status``` - Prints if the server is running or not
```cs2 console``` - Connects to the server console
```cs2 update``` - Updates CS2, Metamod & CounterStrikeSharp to latest versions, then starts the server
```cs2 metamod``` - Updates Metamod to the latest version
```cs2 csharp``` - Updates CounterStrikeSharp to the latest version
```cs2 checkupdate``` - Same as update, but only continues if a newer version of CS2 is available
