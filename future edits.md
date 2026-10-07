Today you don't need to touch Docker in production. Vercel and Railway take your code and package and run it themselves; Railway already turns your worker-host into a container behind the scenes, so in a sense you're already using containers without managing them. You'd use Docker deliberately when one of these situations comes up:

| When | Why Docker becomes necessary | How likely for Sim Trans |
|---|---|---|
| **A client requires the system inside their own network** (government, courts, banks: "nothing leaves our servers") | You deliver SIM.AI as a set of Docker images (or a Kubernetes package) that their IT team runs on their servers | **High.** This is common with UAE government and legal clients, and probably your first real trigger |
| **Data must stay in the UAE** | Move the worker-hosts from Railway to a UAE cloud region, for example Azure UAE North, which runs Docker containers | Medium. It comes up with confidential legal work |
| **The LiveKit bill gets large** | LiveKit is open source: you can run your own LiveKit server as a Docker container on your own cloud machines instead of paying per minute | Later. Worth checking once LiveKit costs reach several hundred dollars a month or more |
| **You outgrow Railway** (cost, limits, or needing auto-scaling for many simultaneous events) | Run the worker-hosts as containers on a larger platform (Azure Container Apps, AWS, Google Cloud Run, Kubernetes) | Later, when the SaaS version grows |
| **A venue box for poor internet** | A small computer at the venue running parts of the system locally, shipped as Docker containers | Possible, if unreliable venue internet keeps causing problems |

**What I'd do now:** nothing urgent, plus one cheap step that keeps your options open. Make sure the worker-host has a proper **Dockerfile**: an exact recipe for its container, including Node version and ffmpeg. Railway can build from it, and the day a client wants the system on their own servers or in the UAE, you already have a portable package rather than a rushed migration. The read-only Docker prompt I gave you checks how Railway currently builds the worker-host. If it says there's no Dockerfile, ask Claude Code:

```
Add a production Dockerfile for the worker-host (multi-stage, pinned Node version, ffmpeg, non-root user, health check) and a .dockerignore, make Railway build from it, and confirm the deployed behaviour is identical. Document in DEPLOY.md how to build and run it on another platform.
```

The dashboard and attendee page can stay on Vercel even in these scenarios. An on-premise client would also get them as containers, but that's a separate project to plan when such a client appears.
