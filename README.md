# System Design of GitHub Repository Events Monitoring Application

## Contents

1. [Task Assignment](#task-assignment)
2. [Architecture](#architecture)
3. [Components Design](#components-design)
   - [GitHub API](#github-api)
     - [Obstacles](#obstacles)
     - [Proposed Approach](#proposed-approach)
     - [Alternatives](#alternatives)
   - [Server](#server)
     - [GitHub API Use](#github-api-use)
     - [Scheduler](#scheduler)
       - [Proposed Approach](#proposed-approach-1)
     - [GitHub API Calls Minimization](#github-api-calls-minimization)
     - [Application API Endpoints](#application-api-endpoints)
       - [API](#api)
       - [Security](#security)
   - [Database](#database)
4. [Deployment](#deployment)
   - [Local Development](#local-development)
   - [Cloud (AWS)](#cloud-aws)

---

## Task Assignment

The objective of this assignment is to track activities on GitHub. To achieve this, utilize the GitHub Events API. The application should monitor up to five configurable repositories and generate statistics based on a rolling window of either 7 days or 500 events, whichever is less. These statistics are made available to end-users via a REST API, showing the average time between consecutive events, separately for each combination of event type and repository name. The application should minimize requests to the GitHub API and retain data through application restarts.

## Architecture

Overall application architecture:

![](img/architecture.svg)

## Components Design

### GitHub API

GitHub API supports different strategies on how to use the API:

- **Unauthenticated Access**
  - Only 60 requests per hour are available.
- **Authenticated Access**
  - 5,000 requests per hour.
  - Different authentication flows (Personal Access Token, GitHub App, OAuth).
  - GitHub Enterprise Cloud organization gives 15,000 requests per hour.

Some demo uses of GitHub API with different flows are presented in [github_api.ipynb](github_api.ipynb).

#### Obstacles

There are several obstacles and limitations when we use the GitHub API:

- There is a limit on API calls from the GitHub API.
- Events are paginated.
- Only events created within the past 90 days are included.
- Only the last approximately 300 events are included (documentation implies exactly 300, but in reality, it is under 300 and not precise).
- Documentation says that the API is not built to serve real-time use cases; depending on the time of day, event latency can be anywhere from 30 seconds to 6 hours.
- Best practices advise not to use concurrent requests (though up to 100 concurrent requests are possible).

#### Proposed Approach

After thorough consideration of each option, the best solution is to use the organizational Personal Access Token flow:

- Create a GitHub organization.
- Create a Personal Access Token for the organization.
- Use authenticated API utilizing resources of the organization.
- Use conditional requests to check if there were new events to avoid rate limit usage.
- If needed, send requests to check API rate limit. Such requests do not affect the rate limit.

#### Alternatives

Other authentication flows can be used, but they require more complexity and are not ideal for our use case:

- **GitHub App**: Should be created by the organization and installed on that organization, requires management of private keys, and uses JWT tokens for authentication, which means we need to monitor the expiration of tokens. Overall, for this application, it is overly complex.
- **OAuth**: Can be used only on behalf of some user to access their resources. Authentication is also very complex for this application.

### Server

There are two main components:

- **Web Server**: We will use the FastAPI Python framework. Like Flask, it is very simple and very useful when we need to write APIs very quickly.
- **Scheduler**: For data gathering, we will use the APScheduler Python framework, which allows us to schedule task execution, like sending requests to the GitHub API.

Both components will run in the same process concurrently, accessing common resources—like modules, database connections, and configurations.

![](img/server_component.svg)

#### GitHub API Use

For our case, we need to use only the following endpoints:

- **`GET /repos/{owner}/{repo}/events`**: Endpoint to get events.
  - **Query Parameters**:
    - `per_page`: Number of results per page.
    - `page`: Page number of the results to fetch.
  - **Header Attributes**:
    - `Authorization: Bearer {organization access token}`
    - `Last-Modified`: Response attribute—when data were last modified.
    - `If-Modified-Since`: Used with `Last-Modified` to make conditional requests and get data only when there are new events. It also can be used to filter older events and save the newest.

- **`GET /rate_limit`**: Get information about our rate limit. Does not increase request usage count.

Example of event schema:

```json
{
  "id": "43756195112",
  "type": "PushEvent",
  "actor": {
    "id": 68920274,
    "login": "eleazar-rivas",
    "display_login": "eleazar-rivas",
    "gravatar_id": "",
    "url": "https://api.github.com/users/eleazar-rivas",
    "avatar_url": "https://avatars.githubusercontent.com/u/68920274?"
  },
  "repo": {
    "id": 879455410,
    "name": "eleazar-rivas/ESET-KeyGen-2024",
    "url": "https://api.github.com/repos/eleazar-rivas/ESET-KeyGen-2024"
  },
  "payload": {
    "repository_id": 879455410,
    "push_id": 21180234309,
    "size": 1,
    "distinct_size": 1,
    "ref": "refs/heads/master",
    "head": "0e3d96550524b41fdace6542aa45d9f8010e6e1b",
    "before": "831a60d3aa9d226bfe9cf67c817c8ba322a8f5d2",
    "commits": [
      {
        "sha": "0e3d96550524b41fdace6542aa45d9f8010e6e1b",
        "author": {
          "email": "68920274+eleazar-rivas@users.noreply.github.com",
          "name": "eleazar-rivas"
        },
        "message": "Commit",
        "distinct": true,
        "url": "https://api.github.com/repos/eleazar-rivas/ESET-KeyGen-2024/commits/0e3d96550524b41fdace6542aa45d9f8010e6e1b"
      }
    ]
  },
  "public": true,
  "created_at": "2024-11-12T20:13:10Z"
}
```

- Events are sorted by `created_at` (FIFO)—the fact that it is sorted is not documented, but it was confirmed empirically in the API demo.
- The `type` field represents the event type: `PushEvent` in the example above. Other examples can be seen in the [documentation](https://docs.github.com/en/rest/using-the-rest-api/github-event-types?apiVersion=2022-11-28).

Some demo uses of the GitHub API can be found here: [github_api.ipynb](github_api.ipynb).

#### Scheduler

We will use Advanced Python Scheduler (APScheduler) to manage data gathering.

##### Proposed Approach

Simplified sequence diagram to show how the scheduler will work:

![](img/schedule_sequence_diagram.svg)

1. **Scheduler Initialization**
   - Get data from the database (active repositories and their last events).
   - Store last events in a window (either last 7 days or 500 events, whichever is less).
   - Compute time differences between consecutive events.
2. **For each active repository, schedule concurrent request call tasks**
   - Tasks are scheduled based on the average time interval between events.
   - After each request, depending on the result, different rescheduling strategies will be used, in accordance with best practices of GitHub API use.

#### GitHub API Calls Minimization

To minimize the number of API calls, we use the following strategy:

- The average time between events is also an estimation of when to plan the next request.
  - It is simple and is a good start (baseline) for this kind of application.
  - It is adaptive.
- We use conditional requests—checking if there are new events before accessing event data.

Later we could try other strategies to reduce the number of API calls even more (which will lead to less resource consumption):

- Use time series analysis:
  - SARIMA models
  - Kalman Filter for online prediction
- Other predictive models.

It is worth mentioning that it is not possible to use webhooks for this application. Webhooks would allow complete minimization of API calls, but to use webhooks, the repository owner would need to give certain permissions to the organization, which is not feasible in our context.

#### Application API Endpoints

- An API will be provided to external users to manipulate repositories, get event data, and event statistics.
- The API tries to be compatible with the GitHub API, so it is more intuitive for other developers to use.
- Also, as a bonus from the FastAPI framework, an endpoint with REST API documentation will be automatically provided.

##### API

- **`GET /api/repos/{owner}/{repo}/events/`**: Get events from the specified repository.
  - **Query Parameters**:
    - `type`: Gets events of the specified type.
    - Other necessary query parameters...
- **`GET /api/repos/{owner}/{repo}/events/stats/`**: Get average time between events from the specified repository.
  - **Query Parameters**:
    - `type`: Gets average time statistics of events of the specified type.
    - Other necessary query parameters...
- **`GET /api/repos/`**: Gets all the repositories.
- **`GET /api/repos/active/`**: Gets all the active repositories.
- **`POST /api/repos/{owner}/{repo}/activate`**: Activate some repositories so the scheduler can gather data for this repository.
- **`POST /api/repos/{owner}/{repo}/deactivate`**: Deactivate some repositories.
- **`POST /api/repos/`**: Add some repositories to the database.
- **`GET /api/docs`**: Shows API documentation.

##### Security

To limit usage of the API to authorized users only, an API Key will be used:

- It is a very simple approach for this kind of application.
- API Key should be used with HTTPS, which can be provided by cloud providers.
- API Key security middleware will be used to filter unauthorized requests.

### Database

The best database for this use case is MongoDB:

- It is excellent for rapid development and simple applications.
- It is very well suited for storing JSON data.
- Different events have different payloads; MongoDB is schema-less, so there will be no problem storing events of any type.

There will be two entities (collections):

- **Repositories**
  - `url`: URL to the repository.
  - `active`: If the repository is active or not.
  - Other fields necessary for implementation.
- **Events**
  - JSON data from GitHub.

## Deployment

### Local Development

For local development, we will use Docker Compose with two images:

- MongoDB
- Our server image, built from a Dockerfile

### Cloud (AWS)

For deployment, we will use AWS:

- GitHub Actions will be used for automatic deployment:
  - Running tests, linters, making other checks
  - Building the image
  - Registering the image with AWS Elastic Container Registry
  - Deploying the image on the cloud (Amazon Elastic Container Service)
- For the database, we will use Amazon DocumentDB (which uses MongoDB underneath)

---