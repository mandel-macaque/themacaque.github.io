# Using GraphQL to perform GitHub searches with C#

I am not the most avid blogger, and I do not plan to be one. However I have decided to write some blogs/notes about programming problems I have solved to which there is not much,
if any, documentation online. This post is going to focus on a small problem I found recently when trying to use [octokit.net](https://github.com/octokit/octokit.net) to find a collection of pull requests and their reviews.

---

## The problem

We want to be able to retrieve all the pull requests created in a repository between two given dates and all their reviews.

## The naive approach: using the REST API

If one wants to interact with the Github API the first thing it does is to search in the [Octokit documentation](http://octokitnet.readthedocs.io/en/latest/.) to find a solution for their needs

The documentation has a number of examples, but to make it easier for the reader I have added the needed code to get you started. Perse the implementation is quite simple and very naive. Retrieving the PRs can be divided in 4 different operations:

1. Create a github client to interact with the GitHub API.
2. Perform a search with the criteria we have using [GitHubs search API](https://octokitnet.readthedocs.io/en/latest/search/#search-pull-requests).
3. Get all the PRs returned by the search.
4. Retrieve all the PR reviews.

There is no more to it than that, lets take a look at the actual implementation:

### Implementation using Octokit

The following snippets are from an actual application I have been writing. Where ever you see a Log.Error/Debug just think about it as a log message (I am using [Serilog.Sinks.Console](https://github.com/serilog/serilog-sinks-console)).

Perform the search.

```csharp
async Task<SearchIssuesResult?> GetPullRequests (string repository, DateTimeOffset startDate, DateTimeOffset endDate)
{
    // we need to use the search api for this:
    try {
        var request = new SearchIssuesRequest {
            Type = IssueTypeQualifier.PullRequest,
            Repos = new () { repository },
            Created = new (startDate, endDate),
        };
        return await ghClient.Search.SearchIssues (request).ConfigureAwait (false); // yes, PRs are issues.. \0/ 
    } catch (Exception e) {
        Log.Debug ("Exception {@Message} searching for pull requests", e.Message);
        return default;
    }
}
```

Once we have received the search results, we need to get the actual PR objects. The search API does not return a PR but a
[SearchIssueResult](https://github.com/octokit/octokit.net/blob/356588288e2d07fb4844911f6e03ef129540b124/Octokit/Models/Response/SearchIssuesResult.cs) which is an object that
represents both, pull requests and issues. As the comment in the code says, GitHub treats PRs as issues. The SearchIssuesResult contains all the numbers of the PRs that
match the search. Ideally at this point, we would be done. We could get all the numbers of the PRs that matched and retrieve them, but there are two important details
that we need to tackle:

1. There is no API in Octokit to get a collection of PRs, you can only retrieve them one by one [source](https://github.com/octokit/octokit.net/blob/b312ae6a050fdef3b5093afe6088782b002c1cd0/Octokit/Clients/PullRequestsClient.cs) and the [criteria object](https://github.com/octokit/octokit.net/blob/b312ae6a050fdef3b5093afe6088782b002c1cd0/Octokit/Models/Request/PullRequestRequest.cs) is not very helpful.
2. The PullRequest object does not have full PullRequestReview objects, it only have a reference to the requested users [source](https://github.com/octokit/octokit.net/blob/b312ae6a050fdef3b5093afe6088782b002c1cd0/Octokit/Models/Response/PullRequest.cs).

The above makes our code a little uglier, yet doable. We can create a single method that will take a PR number, get the PR object AND all of its reviews and returns those in a tuple. Once
we have the method for a single PR, we can create a method that will do the loop in an async way so that all requests are done in parallel.

```csharp
async Task<(PullRequest Request, PullRequestReview [] Reviews)?> GetPullRequest (RepositoryConfig repositoryConfig, int pullRequestId)
{
    try {
        var pullRequest = await ghClient.PullRequest
            .Get (repositoryConfig.Org, repositoryConfig.Project, pullRequestId)
            .ConfigureAwait (false);
        Log.Debug ("SUCCESS retrieving PR {@PRId} in repo {@Org}/{@Project}", pullRequestId, repositoryConfig.Org, repositoryConfig.Project);
        // retrieve the reviews for the PR
        var reviews = await ghClient.PullRequest.Review
            .GetAll (repositoryConfig.Org, repositoryConfig.Project, pullRequestId).ConfigureAwait (false);
        var reviewsArray = reviews.ToArray ();
        Log.Debug ("SUCCESS retrieved {@ReviewCount} reviews for PR {@PRId} in {@Org}/{@Prject}", reviewsArray.Length, pullRequestId, repositoryConfig.Org, repositoryConfig.Project);
        return (Request: pullRequest, Reviews: reviewsArray);
    } catch (Exception e) {
        Log.Error ("Exception {@Message} when trying to retrieve PR {@PRId} in repo {@Org}/{@Project}", e.Message, pullRequestId, repositoryConfig.Org, repositoryConfig.Project);
        return default;
    }
}

Task<(PullRequest Request, PullRequestReview [] Reviewa)? []> GetAllPullRequests (RepositoryConfig repository, IReadOnlyList<int> ids)
{
    var prTasks = new Task<(PullRequest Request, PullRequestReview [] Reviews)?> [ids.Count];
    for (int index = 0; index < prTasks.Length; index++) {
        prTasks [index] = GetPullRequest (repository, ids [index]);
    }
    return Task.WhenAll (prTasks);
}
```

If you look at the above code and wonder why I an using a Task.WhenAll rather than an await per loop instruction, ping me, we need to talk :)

Later in your code you can use the methods as follows:

```csharp
DateTimeOffset startDateTimeOffset = new (startDate.ToDateTime (TimeOnly.MinValue));
DateTimeOffset endDateTimeOffset = new (endDate.ToDateTime (TimeOnly.MaxValue));

var prsSearchTask = GetPullRequests ($"{repository.Org}/{repository.Project}", startDateTimeOffset, endDateTimeOffset);

var searchResult = await prsSearchTask.ConfigureAwait (false);

var prsTask = GetAllPullRequests (repository, prsSearchTask.Result.Items.Select (i => i.Number).ToArray ());
```

### You have exceeded a secondary rate limit...

The above code works perfectly alright, but has a fundamental flaw, it uses *a lot* of requests. And because we are performing the requests in parallel
you will get a secondary threshold warning from GitHub:

> HTTP/2 403
> Content-Type: application/json; charset=utf-8
> Connection: close

> {
>   "message": "You have exceeded a secondary rate limit and have been temporarily blocked from content creation. Please retry your request again later.",
>   "documentation_url": "https://docs.github.com/rest/overview/resources-in-the-rest-api#secondary-rate-limits"
> }

At this point, we need to think about how we are approaching the problem. There are several things we can do to fix the rate problem:

1. Throttle our application. Request the limits of our PAT and adapt to it.
2. Reduce the number of requests.

I am a fan of implementing both solutions. Not because we reduce the number of requests we are going to fix the problem, it will simply take us longer to get
there. Writing code that throttles is not a hard problem to solve, and I am sure there are lots of solutions out there, but how do we reduce the number of
requests if we do not own the GitHub REST API? The best approach is to move away from the REST API and uses [GitHub GraphQL endpoint](https://docs.github.com/en/graphql).

## Solving the problem with GraphQL

We already have seen how to solve the problem with Octokit, but how do we solve the problem using GraphQL, or better, can we find any examples on how to do it? Well,
either my google foo is not good, or there are no examples our there, which is the reason I an writing this post to being with. As we did with the Octokit implementation,
we are going to focus on the steps we need to perform to get to our solution:

1. Find a library that can do must of the work for us.
2. Perform a GraphQL search that will return what we are looking for.
3. Parse the results.

### Calling GraphQL from C#

I am by definition a lazy developers, if there is something out there that already does the job I tend to avoid re-inventing the wheel.
Although there are no libraries that use the GitHub GraphQL endpoint in C# there is a [GraphQL.Client](https://github.com/graphql-dotnet/graphql-client) library for dotnet.
Yes, it does no fully solve our problem but does get us there:

```csharp
var personAndFilmsRequest = new GraphQLRequest {
    Query =@"
    query PersonAndFilms($id: ID) {
        person(id: $id) {
            name
            filmConnection {
                films {
                    title
                }
            }
        }
    }",
    OperationName = "PersonAndFilms",
    Variables = new {
        id = "cGVvcGxlOjE="
    }
};
```

The above is taken from their examples, we will see how we integrate that with our code alter.

### Using GitHubs GraphQL endpoint

There is no point of me getting into describing how GraphQL works because I am no expert (I have read the docs and used it, but no expert).

The best way for you to understand GraphQL is to use the [Explorer provided by GitHub](https://docs.github.com/en/graphql/overview/explorer). To make
your life easier, you can use the following query as a starting point:

Query:

```json
query GetPRs($query: String!, $cursor: String) { 
    search(query:$query type:ISSUE after:$cursor first:100) { 
        issueCount
        edges { 
            cursor
            node { 
                ... on PullRequest { 
                    author {
                        ... on User { 
                            login 
                            name  
                        } 
                    }
                    authorAssociation
                    number 
                    title 
                    closed 
                    createdAt 
                    closedAt
                    headRefName
                    isDraft 
                    labels (last:100) { 
                        totalCount 
                        edges { 
                            node { 
                                ... on Label { 
                                    name  
                                } 
                            }
                        }
                    }
                    mergeable
                    merged 
                    mergedAt 
                    reviewRequests (last:100) { 
                        totalCount 
                    } 
                    reviews (last:100) { 
                        totalCount 
                        edges { 
                            node { 
                                ... on PullRequestReview { 
                                    state 
                                    commit {
                                        id
                                    }
                                    author { 
                                        ... on User { 
                                            login 
                                            name
                                        } 
                                    }
                                } 
                            } 
                        } 
                    } 
                    state
                    updatedAt
                } 
            } 
        }  
    } 
}

```

Variables (you need to update the values of startDateString, endDateString and repository):

```json
{
    "query": "created:{startDateString}..{endDateString} is:pr repo:{repository}"
}
```

Now, GraphQL queries return json, but you need to understand several concepts to understand the result:

* GraphQL allows to fet objects
* GraphQL provides a way to go through the connections.
* A connection is a collection of objects
* pageInfo contains information about the pagination of the results.
* An array of records is returned in an edge.
* An edge has nodes and a cursor

The above query returns a result similar to the following (this is from one of our open source repos, do not worry):

```json
{
  "data": {
    "search": {
      "issueCount": 58,
      "edges": [
        {
          "cursor": "Y3Vyc29yOjE=",
          "node": {
            "author": {
              "login": "dustin-wojciechowski",
              "name": null
            },
            "authorAssociation": "MEMBER",
            "number": 17073,
            "title": "[src] Added try catches to the methods in AVAudioPlayer to prevent Initiali…",
            "closed": false,
            "createdAt": "2022-12-15T18:46:40Z",
            "closedAt": null,
            "isDraft": true,
            "labels": {
              "totalCount": 0,
              "edges": []
            },
            "mergeable": "MERGEABLE",
            "merged": false,
            "mergedAt": null,
            "reviewRequests": {
              "totalCount": 0
            },
            "reviews": {
              "totalCount": 0,
              "edges": []
            },
            "state": "OPEN",
            "updatedAt": "2022-12-15T20:42:01Z"
          }
        },
        {
          "cursor": "Y3Vyc29yOjI=",
          "node": {
            "author": {},
            "authorAssociation": "CONTRIBUTOR",
            "number": 17072,
            "title": "[net8.0] Update dependencies from xamarin/xamarin-macios",
            "closed": false,
            "createdAt": "2022-12-15T17:59:36Z",
            "closedAt": null,
            "isDraft": false,
            "labels": {
              "totalCount": 0,
              "edges": []
            },
            "mergeable": "MERGEABLE",
            "merged": false,
            "mergedAt": null,
            "reviewRequests": {
              "totalCount": 0
            },
            "reviews": {
              "totalCount": 0,
              "edges": []
            },
            "state": "OPEN",
            "updatedAt": "2022-12-15T18:02:32Z"
          }
        },
       ...
    }
  }
}
```

At this point, we are close to have the full solution. We need to solve two problems:

* Deserializing edges and nodes.
* Deal with pagination.

### Deserializing GraphQL responses

As I mentioned, GraphQL returns json objects, so we can use System.Text.Json to deal with the result, which is what GraphQL.Client uses. But
System.Text.Json does now know how to handle Edges and Nodes :/ 

The following generic code will help you with that, we just need an Edge and a Node class that will contain the needed information, hen just use
the generic type you need to deserialize:

```csharp
public record GraphQLNode<T> {

	[JsonPropertyName ("node")]
	public T? Node { get; set; }

	[JsonPropertyName ("cursor")]
	public string? Cursor { get; set; }
}

public record GraphQLEdge<T> {
	[JsonPropertyName ("edges")]
	public GraphQLNode<T> [] Edges { get; set; } = Array.Empty<GraphQLNode<T>> ();

	[JsonPropertyName ("totalCount")]
	public int TotalCount { get; set; }
}
```

### Pagination

Pagination is a pain in any language, any framework in any place.. but it has to be done and your users will be happy about it. Now, how do we deal with 
pagination in the GraphQL world and GitHub. If you go back to the qeury sample I provided you will notices that I am using a variable called *$cursor*:

```json
query GetPRs($query: String!, $cursor: String) { 
    search(query:$query type:ISSUE after:$cursor first:100) { 
    }
}
```

This cursor is what we are going to use to let GitHub know that we want the first 100 results after a given position. 100 is not a random number, is the maximun number of results
allowed by GitHub. With that in mind, the process to follow is vry very simple:

1. Perform the first request with no cursor in the variables json.
2. Receive the response, deserialize it and check the response.
3. If TotalCount is bigger than 100, get the *LAST* node cursor and perform a second request.

It is very important to pay attention and remember that you need the last node cursor, else you will get duplicated results or
if you use the first one, an infinite loop.

The following code uses a generic, gut shows the process that needs to be performed. I am use dotnet 7 so my interfaces have static methods, all
the rest is straight forward for the above described steps.

```csharp
public async Task<IEnumerable<TReturn>> GetGraphQLSearch<TSearch, TReturn> (string repository, DateOnly startDate, DateOnly endDate) where TSearch : IGraphQLSearch<TReturn>
{
    try {
        string? cursor = null;
        int? issuesCount = null;
        int currentlyFetch = 0;
        List<TReturn> result = new ();
        do {
            var graphQlRequest = (cursor is null)
                ? TSearch.BuildRequest (repository, startDate, endDate)
                : TSearch.BuildRequest (repository, startDate, endDate, cursor);
            var graphQLResponse = await graphQLClient.SendQueryAsync<TSearch> (graphQlRequest);
            if (graphQLResponse.Errors is not null) {
                var errors = string.Join (',', graphQLResponse.Errors.Select (e => e.Message));
                Log.Error ("GraphQL Errors {@GraphQLErrors}", errors);
                return Array.Empty<TReturn> ();
            }

            if (graphQLResponse.Data.Search is null)
                return Array.Empty<TReturn> ();

            // fluent and nullability are not friends :(
            var nodes = from node in graphQLResponse.Data.Search.Edges
                        where node is not null
                        select node.Node;

            var nodesArray = nodes.ToArray ();
            result.AddRange (nodesArray);
            Log.Debug ("Received {@PRCount} from GraphQL search", nodesArray.Length);

            if (issuesCount is null) {
                Log.Debug ("GraphQL first search request for {@Repository}", repository);
                // the issue count will be the TOTAL issue count (in the first request) minus what we have in the
                // current request
                issuesCount = graphQLResponse.Data.Search.IssueCount;
                Log.Debug (
                    "GraphQL search issue count for {@Repository} is {@IssueCount} after removing {@ResponseCount}",
                    repository, issuesCount, graphQLResponse.Data.Search.Edges.Length);
            }

            currentlyFetch += graphQLResponse.Data.Search.Edges.Length;
            Log.Debug ("GraphQL search fetched {@FetchCount} out of {@IssueCount}", currentlyFetch, issuesCount);

            // if the issuesCount is not 0, set the cursor to do a second request
            if (issuesCount > currentlyFetch) {
                Log.Debug ("GraphQL search remaining issues {@IssueCount}", issuesCount);
                cursor = graphQLResponse.Data.Search.Edges [^1].Cursor; // grab the last cursor
            } else {
                // we hve retrieved all pages, could be achieved by setting cursor to null, but always be
                // explicit over implicit
                Log.Debug ("GraphQL search retrieved all {@PRCount} PRs", issuesCount);
                break;
            }
        } while (cursor is not null);

        return result;
    } catch (Exception e) {
        Log.Debug ("Exception GraphQl search {@ExceptionMessage}", e.Message);
        return Array.Empty<TReturn> ();
    }
}

```

### Testing

The last step to have a real solution is to be able to write a test. But how do we write tests against a GRaphQL API? Well, believe it or not is not 
as hard as it sounds. T o do show what we are going to be doing is to use [RichardSzalay.MockHttp](https://github.com/richardszalay/mockhttp) a library that can only be said to be the nest best thing
after slice bread (for testing network calls).

The MockHttp is the corner stone to test all the different calls. What we want to do is to create a GraphQL.Client client that uses an HttpClient that has a
mocked handler, once we have done that, we can set the expectations, and happy testing (the sample code uses xUnit):

```csharp
public class TestGraphQL {
    readonly HttpClient httpClient;
	readonly GraphQLHttpClient graphQlHttpClient;
	readonly MockHttpMessageHandler mockHttpMessageHandler;
	readonly string githubGraphQLEndpoint = "https://api.github.com/graphql";

    public TestGraphQL () {
        mockHttpMessageHandler = new ();
		httpClient = new (mockHttpMessageHandler);
        // this is where the magic happens
		var options = new GraphQLHttpClientOptions { EndPoint = new (githubGraphQLEndpoint) };
		graphQlHttpClient = new (options, new SystemTextJsonSerializer (), httpClient);
    }

    [Fact]
    public async Task SampleTest ()
    {
        // set the mocker expectations
        var prData = TestsHelpers.ReadJsonFile ("graphQLPRSearch.json");
        Assert.NotNull (prData);
        mockHttpMessageHandler.When (githubGraphQLEndpoint)
            .WithPartialContent ("GetPRs")
            .Respond ("application/json", prData);

        // tet your code :) 
    }
}
```

From the above I would like to point out a few things:

1. You need to create the http client using the mock handler.
2. You can not only return data, but return diff reponses. Read the docs
3. The mock handler knows what that to return because I am using the `WithPartialContent` method with the function name of my query `GetPRs`. If you have more queries, keep that in mind. 


## Conclusion

With the given exmaples, you should have all the pieces you need to perform queries against the GitHub GraphQL api using C#. In is not complicated, but there is no documentation or many example. Hope it helps.