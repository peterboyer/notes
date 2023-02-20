Run `playwright` (in headed/non-headless mode) using the
`mcr.microsoft.com/playwright:focal` Docker image on a non-CI machine.

Why:

- Avoids installing browser drivers to local machine (just compartmentalise the
  environment with a container)
  - or if it's too difficult on your machine's os/distro.

```
Looks like you launched a headed browser without having a XServer running.
Set either 'headless: true' or use 'xvfb-run <your-playwright-app>' before running Playwright.

[pid=___][err] [308:321:0220/075417.119909:ERROR:bus.cc(399)] Failed to connect to the bus: Failed to connect to socket /var/run/dbus/system_bus_socket: No such file or directory
[pid=___][err] [308:308:0220/075417.121903:ERROR:ozone_platform_x11.cc(238)] Missing X server or $DISPLAY
[pid=___][err] [308:308:0220/075417.121911:ERROR:env.cc(255)] The platform failed to initialize.  Exiting.
```

Usage:

- `yarn test` **outside** the container will start `docker run ...`
- `yarn test` **inside** the container will run `playwright ...`

Breakdown:

- `--user "$(id -u)"` any created files created in the container (e.g.
  `node_modules/.vite`) will be owned by the host user.
- `--volume $PWD/..:/app` mount the app directory to use the docker container
  as an execution environment only
- `--workdir /app` enter the interactive environment already cd'ed into the
  `/app` volume directory (your $PWD / project dir)
- `--env CONTAINER=1` flag to toggle between `yarn test` behaviours in/out of
  the container
- `--env DISPLAY=$DISPLAY` forward the display environment variable for xserver
  to render headed browser to host's desktop
- `xhost + local:docker > /dev/null` run on the host to allow x server to
  accept x server comms from the local docker's host

## `./scripts/test`

```bash
#!/bin/bash

if [[ $CONTAINER == 1 ]]; then
	yarn playwright test $@
else
	xhost + local:docker > /dev/null
	docker run \
		-it \
		--rm \
		--ipc=host \
		--net=host \
		--user "$(id -u)" \
		--privileged \
		--volume $PWD/..:/app \
		--workdir /app \
		--env CONTAINER=1 \
		--env DISPLAY=$DISPLAY \
		mcr.microsoft.com/playwright:focal \
		/bin/bash
fi
```

If it's still not working try adding these lines to the `docker run` command:

```bash
		--volume $HOME/.Xauthority:/root/.Xauthority:rw \
		--volume /tmp/.X11-unix:/tmp/.X11-unix \
```

- `--volume $HOME/.Xauthority:/root/.Xauthority:rw` mount host's xauthority
  home file to communicate with host's desktop
- `--volume /tmp/.X11-unix:/tmp/.X11-unix` forward the x server's protocol
  socket to the host's desktop's x server

## `./package.json`

```json
{
	"scripts": {
		"test": "scripts/test",
		"test:debug": "scripts/test --debug"
	},
	"dependencies": {
	},
	"devDependencies": {
		"@playwright/test": "^1.30.0"
	}
}
```
