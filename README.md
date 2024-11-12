# System Design of GitHub Repository Events Monitoring Application

TODO - HERE PUT THE CONTENTS 

## Architecture

Overall application architecture:

![](img/architecture.svg)

## Components Design

### GitHub API

Github API supports different strategies on how to use the API:

- Unauthenticated access
  - Only 60 requests per hour are available.
- Authenticated access
  - 5000 requests per hour.
  - Different authentication flows (Personal Access Token, GitHub App, OAuth)
  - GitHub Enterprise Cloud organization gives 15000 requests per hour.

Some demo uses of GitHub API with different flows are presented in [github_api.ipynb](github_api.ipynb).

#### Obstacles

There are several obstacles and limitations when we use GitHub API:
- There is a limit on API calls from the GitHub API.
- Events are paginated.1
- Only events created withing the past 90 days are included.
- Only last $\approx$ 300 events are included (documentation implies exactly 300, but in reality it is under 300 and not precise).
- Documentation says, that API is not build to server real-time use cases, depending on the time of day, event latency can be anywhere from 30s to 6h.
- Best practices advices to not use concurrent requests (though up 100 concurrent requests if possible).

#### Proposed approach

After thorough consideration of each option, the best solution will be:
- Create GitHub organization.
- Create Personal Access Token for the organization.
- Use authenticated API utilizing resources of organization.
- Use conditional requests to check if there were events to avoid requests usage.
- If needed, send requests to check API rate limit. Such requests does not affect rate limit.


TODO - IF THERE IS A TIME, BUSINESS ACTIVITY DIAGRAM HERE PLEASE

#### Alternatives

Other authentication flows can be used, but they require more complexity and are not ideal for our use case:

- GitHub App - should be created by organization and installed on that organization, requires management of private keys, and uses JWT tokens for authentication, which means we need to monitor expiration of tokens. Overall for this application it has an overkill complexity.
- OAuth - can be used only on behalf of some user to take his resources. Authentication is also very complex for this application. 

### Server

There are 2 main components:

- As a web server we will use FastAPI python framework. Like Flask, it is very simple and very useful when we need to write API very fast.
- For data gathering we will use APSchedule python framework, which allows to plan tasks execution, like sending requests to GitHub API.

Both components will run in the same process concurrently, accessing common resources - like modules, database connections and configurations.

![](img/server_component.svg)

#### GitHub API use

For our case we need to use only following endpoints:

- `GET /repos/{owner}/{repo}/events` - endpoint to get events.
  - Query parameters:
    - `per_page` - number of results per page.
    - `page` - page number of the results to fetch.
  - Header attributes:
    - `Authentication: Bearer {organization access token}`
    - `last-modified` - response attribute - when data were last modified.
    - `if-modified-since` - used with last-modified to make conditional request, and get data only when there are new events. It also can be used to filter older events and save the newest.

- `GET /rate_limit` - get information about our rate limit. Does not increase requests use count.

Example of event scheme:
```json
{'id': '43756195112',
 'type': 'PushEvent',
 'actor': {'id': 68920274,
  'login': 'eleazar-rivas',
  'display_login': 'eleazar-rivas',
  'gravatar_id': '',
  'url': 'https://api.github.com/users/eleazar-rivas',
  'avatar_url': 'https://avatars.githubusercontent.com/u/68920274?'},
 'repo': {'id': 879455410,
  'name': 'eleazar-rivas/ESET-KeyGen-2024',
  'url': 'https://api.github.com/repos/eleazar-rivas/ESET-KeyGen-2024'},
 'payload': {'repository_id': 879455410,
  'push_id': 21180234309,
  'size': 1,
  'distinct_size': 1,
  'ref': 'refs/heads/master',
  'head': '0e3d96550524b41fdace6542aa45d9f8010e6e1b',
  'before': '831a60d3aa9d226bfe9cf67c817c8ba322a8f5d2',
  'commits': [{'sha': '0e3d96550524b41fdace6542aa45d9f8010e6e1b',
    'author': {'email': '68920274+eleazar-rivas@users.noreply.github.com',
     'name': 'eleazar-rivas'},
    'message': 'Commit',
    'distinct': True,
    'url': 'https://api.github.com/repos/eleazar-rivas/ESET-KeyGen-2024/commits/0e3d96550524b41fdace6542aa45d9f8010e6e1b'}]},
 'public': True,
 'created_at': '2024-11-12T20:13:10Z'}
```

- Events are sorted by `created_at` (FIFO) - the fact that it is sorted is not documented, but it was confirmed empirically in the api demo.
- `type` field represents event type: `PushEvent` in the example above. Other example can be seen here in [documentation](https://docs.github.com/en/rest/using-the-rest-api/github-event-types?apiVersion=2022-11-28).

Some demo uses of GitHub API can be found here [github_api.ipynb](github_api.ipynb).

#### Scheduler



##### Proposed approach
TODO - how the proposed algorithm can work
Explain your choice, maybe some pseudocode or diagram to express idea of how it will work

##### Other approaches

Here write down all important alternatives you have studied

#### Application API Endpoints

TODO - give some info here, pls. But really fast.

### Database

#### Proposed - MongoDB

Schema - design of the persistence.

Argument here why and what

#### Other databases

## Deployment

Here you will put deployment diagram scheme

### Local Development

## Security

Where security tokens will be stored and how they will be deployed.
