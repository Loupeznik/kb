# Services

## Adding a basic Systemd service

Create a file `/etc/systemd/system/my-service.service`

```toml
[Unit]
Description=MY SERVICE

[Service]
Type=simple
WorkingDirectory=/opt/apps/myapp # working directory for the service
ExecStart=/opt/apps/myapp/bin/myapp # path to the executable
User=ubuntu # user to run the service
Usergroup=ubuntu # group to use when running the service
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## Adding a Systemd service for a .NET application

Run `sudo vi /etc/systemd/system/ts3viewer2.service`

```toml
[Unit]
Description=TS3Viewer2 API

[Service]
WorkingDirectory=/opt/ts3viewer2/api
ExecStart=/usr/bin/dotnet /opt/ts3viewer2/api/DZarsky.TS3Viewer2.Api.dll
Restart=always
RestartSec=10
SyslogIdentifier=ts3viewer2-api
User=ubuntu
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```
