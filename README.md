# valetudo-crontab
## A workaround for not working timers
I am running a dustcloud build on my roborock s6 vacuum bot and on valetudo 0.6.1 the timers do not work.
So this is a little hacky workaround to set up cronjobs for scheduling roborock s6 cleaning tasks, by calling valetudos REST API endpoints.

Be aware that the api endpoints might be different for you than in my examples if you are running another version of valetudo. These are not documented, as they 
are intended to be used by the WebGUI only, so better don't bother the devs about it. You are actually meant to use the mqtt API for automation. 

## Setting the correct date and time first
For this to work, the correct date and time has to be set on the robot. So check by running `date`. If the datetime isn't correct, have a look inside
`/opt/roborock/watchdog/ntpserver.conf` and replace any IPs in there with only the IP of your router (if that is acting as your local ntp server). 
If you cannot set your local router as an ntp server for whatever reason, you can enter another ntp server `server 0.somentpsomewhere.com` instead.
If you have set strict iptables rules and the bot cannot get outside of your LAN, you can probably set the robots date and time without ntp locally by running
`date -s '2021-01-20 10:15:00'` (or whatever the required dateformat is) to set the systemclock and then `hwclock -w` to write the system time to the
hardwareclock.

## Finding out what API endpoints to call
First I would recommend to take a look at the WebGUI of your valetudo installation in the browser, and take a peek into the
developer console. Then navigate to the sources tab and locate the /services/api.services.js file to check out the available endpoints.
With these api endpoints we can easily set up a schedule on the robot itself. Take note of the request method, it can be GET or PUT with arguments passed as json,
in some cases a POST or DELETE is used. If you have to pass arguments as json, you might be lucky and find a way to derive what the keys have to be called and 
what type the values need to be.

We will call these API endpoints with curl, a cli tool to send GET/PUT/POST requests that is already installed on the robot.

## curl API call examples
Take a look at the source like described above to amend the urls accordingly! 

Play around with curl while connected through ssh on the robot, the api will answer you with error messages if you screwed up or if the json data you send is not
structured in the correct way. When a command is accepted, it well respond with OK 200

Starting a full cleaning session:
```
curl --request PUT --url http://127.0.0.1/api/start_cleaning
```

Stopping any jobs the vacuum bot is doing:
```
curl --request PUT --url http://127.0.0.1/api/stop_cleaning
```

Sending the vacuum bot back home:
```
curl --request PUT --url http://127.0.0.1/api/drive_home
```

Get status information of the robot:
```
curl --request GET --url http://127.0.0.1/api/state
```

Get zone information as json if you want to get crazy with scripting or something:
```
curl --request GET --url http://127.0.0.1/api/zones
```

Clean zone id -1 and -4:
```
curl -H "Content-Type: application/json" -d '[-1, -4]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id
```

To find the ids of your map zones open http://YOUR.ROBOT.IP.XX/api/zones in your browser and your will get a response from the API with 
a list of the zones you have configured on your map and their meta-data.

## Scheduling these API calls with cron
Since every linux comes with a native scheduler called cron, we can leverage this to run our own timers without the webgui of valetudo.
First we need to connect to the robot via ssh and create the directory '/etc/crontabs'.
Normally we should run 'crontab -e' to edit our users crontab with vim, but since lots of people have issues using it, just create a new file 
called 'root' in the /etc/crontabs dir, our root user will be running the jobs.

## Better use a cron generator if you don't know the syntax
Or you might end up telling your vacuum robot to start cleaning at 4:00am when you are trying to sleep, or even worse, 
tell it to start a cleaning job every 5 seconds because of a typo. I used this website just to be safe: https://crontab-generator.org/

## crontab examples
Here are some examples from my crontab that you can paste, be sure to understand how the syntax works though 
and test your commands before you make a new one to add.

```
# /etc/crontabs/root
# monday and thursday at 9am clean zones -3 and -2
0 9 * * 1 curl -H "Content-Type: application/json" -d '[-3,-2]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
0 9 * * 4 curl -H "Content-Type: application/json" -d '[-3,-2]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
# start at full clean sundays at noon
0 12 * * 0 curl --request PUT --url http://127.0.0.1/api/start_cleaning >/dev/null 2>&1
# clean zone -8 (pet area) sundays at 18:00
0 18 * * 0 curl -H "Content-Type: application/json" -d '[-8]' --request PUT --url http://127.0.0.1/api/start_cleaning_zones_by_id >/dev/null 2>&1
```

## Start crond so that the scheduler is actually running
Now that you have your crontab prepared, it's time to start the timers :)
Run `crond` manually if you just to want to test something. In order to make it permanent and start the scheduler on boot, edit the `/init/S11valetudo` script
and add this snippet of code
```    
if pgrep -x crond >/dev/null
then
    echo "cron already running"
else
    crond -b
fi
``` 
to the `load()` method like so
```
load() {
    curtime=`cat /proc/uptime | awk -F ' ' '{print $1}'`
    echo "[$curtime] start valetudo-daemon"
    start-stop-daemon -S -b -q -m -p /var/run/valetudo-daemon.pid -x /usr/local/bin/valetudo-daemon.sh
    load() {
    curtime=`cat /proc/uptime | awk -F ' ' '{print $1}'`
    echo "[$curtime] start valetudo-daemon"
    start-stop-daemon -S -b -q -m -p /var/run/valetudo-daemon.pid -x /usr/local/bin/valetudo-daemon.sh
    
    # this checks if crond is already running and starts it if not, we only need one instance
    if pgrep -x crond >/dev/null
    then
        echo "cron already running"
    else
        crond -b
    fi
    # end crond section
}

}
```
This way we can ensure that everytime valetudo is spun up, crond is also running in the background.


Hopefully this will help some people who want to set up a cleaning schedule without a smart home system to control their bot.
Have fun and don't blame me if your robovac eats you at night, because you did not rtfm! ;)
