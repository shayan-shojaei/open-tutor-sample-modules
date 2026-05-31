# Open Tutor Sample Modules

Sample module collection for [Open Tutor](https://github.com/shayan-shojaei/open-tutor) — a self-hosted, interactive learning environment.

## Included Courses

| Course | Description |
|--------|-------------|
| **Calculus 3** | Multivariable calculus covering vectors, partial derivatives, multiple integrals, and vector calculus |
| **Building a Load Balancer in Go** | Build a production-grade round-robin HTTP load balancer from scratch using Go's core systems primitives |

## Add These Modules to Open Tutor

```bash
# Register this repo
tutor repo add https://github.com/shayan-shojaei/open-tutor-sample-modules

# Install the Calculus 3 course
tutor module install open-tutor-sample-modules calc3

# Install the Go Load Balancer course
tutor module install open-tutor-sample-modules go-load-balancer
```

Or install both at once:

```bash
tutor repo add https://github.com/shayan-shojaei/open-tutor-sample-modules
tutor module install open-tutor-sample-modules calc3
tutor module install open-tutor-sample-modules go-load-balancer
```

Then start Open Tutor:

```bash
tutor start   # http://localhost:3000
```

## Creating Your Own Module Collection

Fork this repo or start fresh — see the [Open Tutor README](https://github.com/shayan-shojaei/open-tutor#sharing-your-own-modules) for the full guide on creating and sharing modules.
