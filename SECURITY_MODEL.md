# Threat model: Luomin0216/Snake2

_Status: drafted from the code for Devin security scans. Not yet confirmed by the maintainer; facts they confirm go under "User-provided facts"._

## System overview
A single-player Snake game written in Java Swing/AWT (~340 lines in `src/`). It runs locally as a desktop app (`java -jar SnakeGame.jar`), or on macOS inside Docker, drawing to the host's XQuartz X11 server. It has no network listeners, file I/O, persistence, authentication, user accounts or third-party Java dependencies.

## Components
| Name | Code | Notes |
|---|---|---|
| **Game Client** | `src/*.java` (`Main`, `Window`, `ThreadsController`, `KeyboardListener`, `DataOfSquare`, `SquarePanel`, `Tuple`) | Swing UI and the game-loop thread. Its only input is local keyboard events (arrow-key codes 37–40). Game state is shared through static fields (`ThreadsController.directionSnake`, `Window.Grid`) between the AWT event thread and the game thread. |
| **Distributed JAR** | `SnakeGame.jar` (committed binary) | Eclipse Jar-in-Jar loader build from 2014. Its classes (`ControlleurThreads`, `KeyboardListenner`, `ThreadSquare`) **do not match the current `src/`**, so the shipped artifact can't be reproduced from source. README tells users to download and run it directly. |
| **Container & X11 Setup** | `Dockerfile`, `docker-compose.yml`, README "Mac OSX with Docker" steps | Base image `openjdk:17-jdk-slim` (deprecated/unmaintained image line). Compose mounts the whole repo at `/app` (overwriting the image's copied jar), mounts `/tmp/.X11-unix`, sets `DISPLAY=host.docker.internal:0`, adds `cap_add: SYS_ADMIN` and runs as root. Its command references `/app/Snakegame.jar`, which has the wrong case. README tells users to enable "Allow connections from network clients" in XQuartz and run `xhost +`. |

## Trust boundaries & entry points
- **Player → Game Client:** local keyboard only, at the same privilege as the process. No trust boundary is crossed.
- **Publisher/repo → player's machine:** users run the committed `SnakeGame.jar`, or build a container from a public base image. This is the main boundary: whatever is in the jar or the image runs with the user's privileges.
- **Container → host:** the X11 socket/TCP display plus the SYS_ADMIN capability and root user weaken container isolation.
- **Network → host X server:** following the README (`xhost +`, network clients allowed) exposes the Mac's X server to the network. That allows keystroke capture, screen capture and input injection by anyone who can reach it.

## Assets
- Integrity of the user's host machine and session (code execution, keystrokes, screen contents).
- Integrity of the distributed `SnakeGame.jar`.
- The game itself handles no sensitive data, credentials or PII.

## Attacker profiles (draft)
1. **Supply-chain attacker:** tampers with the committed jar, or the base image / its packages.
2. **LAN attacker:** reaches an X server opened by the README's `xhost +` instructions.
3. **Malicious/compromised container process:** tries to escape, using SYS_ADMIN, root and the X11 socket.
4. The local player is not an attacker. They already control the process.

## Existing controls
- No network surface and no parsing of untrusted data in the Java code.
- Input is limited to four key codes; everything else is ignored.
- MIT-licensed public code; Dockerfile installs a minimal set of X libraries and cleans the apt lists.

## Out of scope / accepted risk
_None stated yet._

## User-provided facts
_None yet._

## Open questions
- Is the Docker/XQuartz path actually used, or only the plain `java -jar` path?
- Should game-logic robustness issues (thread-safety races on the static `directionSnake`, infinite `stopTheGame` loop, unsynchronized Swing updates off the EDT) count as security findings or as accepted risk?
- Is `SnakeGame.jar` still the artifact people download (e.g. from blog posts linking the repo)?
