# Twelve Factors #

## 1. Codebase ##
- Having single codebase for your application. Multiple developers can collaberate on single codebase using tools like Git.
- Multiple apps sharing the same code is a violation of twelve-factor i.e. the apps should be separated into their individual codebases. 

## 2. Dependencies ##
- Explicitly declare and isolate dependencies. A twelve-factor app never relies on implicit existence of system-wide packages.
- For eg - in Python, we have `requirements.txt` file that defines the set of dependencies we need to launch the app.
- For isolating dependencies based on different environments, we have `Virtual Environments (venv)` in Python.
- For system-related dependencies (like `curl`), we can use docker containers. In general, we use docker containers to define explicit dependencies and isolate them.

## 3. Config ##
- Move the dynamic configurations like `host`, `port` to a separate config file.
- Python uses `.env` file where we can specify the environment variables and the load then into the app using `os.getenv("HOST")`
- This allows us to use different configurations for different deployment infrastructures - dev, staging, prod.

## 4. Backing Services ##
- Treat backing services as attached resources.

## 5. Build, release, run ##
- The Twelve-factor app uses strict separation between the build, release and run stages.

## 6. Processes ##
- Twelve-factor processes are stateless and share nothing.
- Sticky sessions are a violation of twelve-factor and should never be used or relied upon. 
- Therefore, to store information like `visitorCount` across multiple instances of app, we can use an external backing service (database or caching) which can accessed by all the instances.

## 7. Port Binding ##
- Multiple instances of the app can run on different ports on the same server.
- The twelve-factor app is completely self-contained

## 8. Concurrency ##
- Scale horizontally and apply load balancer between the users and the processes.

## 9. Disposability ##
- The twelve-factor app's processes are disposable, meaning they can be started or stopped at a moment's notice.
- Processes should shutdown gracefully when they receive a `SIGTERM` signal from the process manager.

## 10. Dev/prod parity ##
- The twelve-factor app is designed for continuous deployment by keeping the gap between development and production small.
- The twelve-factor developer resists the urge to use different backing services between development and production. Both developers and operations have access to the prod environment and dev & prod uses same backing services.

## 11. Logs ##
- A twelve-factor app never concerns itself with routing or storage of its output stream.
- Store logs in a centralized location in a structured format, eg - `ELK stack`, `fluentd`, `splunk` etc.

## 12. Admin Processes ##
- Administrative tasks (such as database migrations) should be kept separate from app's processes and they should be running on identical setup and be automated, scalable and reporducible.