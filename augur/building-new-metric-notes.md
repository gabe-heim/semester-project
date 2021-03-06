
1. augur/augur/datasources/<directory for data source>
	1. example: ghtorrent
	2. example file 1: ghtorrent.py
		- create a new function that has the sql query to the database, or API call to GitHub or whatever. But if you're in `ghtorrent.py` (or `facade.py`), its a sql query. Here's an example breakdown: 
            - `@annotate(tag='code-review-iteration') makes the function visible in metrics-status
            - `def code_review_iteration..` is the name of the function called in `routes.py` **in the same data source folder** (which is `augur/augur/datasources/ghtorrent` in our example)

```python 
@annotate(tag='code-review-iteration')
        def code_review_iteration(self, owner, repo=None):
        """
        Timeseries of the count of iterations (being closed and reopened) that a merge request (code review) goes through until it is finally merged

        :param owner: The name of the project owner or the id of the project in the projects table of the project in the projects table. Use repoid() to get this.
        :param repo: The name of the repo. Unneeded if repository id was passed as owner.
        :return: DataFrame with iterations/issue for each issue that week
        """
        repoid = self.repoid(owner, repo)

        codeReviewIterationSQL = s.sql.text("""
        SELECT
            DATE(issues.created_at) AS "created_at",
            DATE(pull_request_history.created_at) AS "merged_at",
            issues.issue_id AS "issue_id",
            pull_request_history.pull_request_id AS "pull_request_id",
            pull_request_history.action AS "action",
            COUNT(CASE WHEN action = "closed" THEN 1 ELSE NULL END) AS "iterations"
        FROM issues, pull_request_history
        WHERE find_in_set(pull_request_history.action, "closed,merged")>0
        AND pull_request_history.pull_request_id IN(
            SELECT pull_request_id
            FROM pull_request_history
            WHERE pull_request_history.action = "closed")   #go by reopened or closed??? (min: completed 1 iteration and has started another OR min: completed 1 iteration)
        AND pull_request_history.pull_request_id = issues.issue_id
        AND issues.pull_request = 1
        AND issues.repo_id = :repoid
        GROUP BY YEARWEEK(issues.created_at) #YEARWEEK to get (iterations (all PRs in repo) / week) instead of (iterations / PR)?
        """)

        df = pd.read_sql(codeReviewIterationSQL, self.db, params={"repoid": str(repoid)})
        return pd.DataFrame({'date': df['created_at'], 'iterations': df['iterations']})
```


3. example file 2: `routes.py` in the same directory, `augur/augur/datasources/ghtorrent/`
    - `server.addTimeseries` (most of the below is annotation. The very last line is what makes it actually work.);;;;

```python
    """
    @api {get} /:owner/:repo/timeseries/code_review_iteration Code Review Iteration
    @apiName code-review-iteration
    @apiGroup Growth-Maturity-Decline
    @apiDescription <a href="com/chaoss/metrics/blob/master/activity-metrics/code-review-iteration.md">CHAOSS Metric Definition</a>. Source: <a href="http://ghtorrent.org/">GHTorrent</a>

    @apiParam {String} owner Username of the owner of the GitHub repository
    @apiParam {String} repo Name of the GitHub repository

    @apiSuccessExample {json} Success-Response:
                        [
                            {
                                "date": "2012-05-16T00:00:00.000Z",
                                "iterations": 2
                            },
                            {
                                "date": "2012-05-16T00:00:00.000Z",
                                "iterations": 1
                            }
                        ]
    """
    server.addTimeseries(ghtorrent.code_review_iteration, 'code_review_iteration')

```

4. example file 3: 'augurAPI.js' in the `augur/frontend/app/` directory needs to have the the metric from `routes.py` mapped to an API endpoint that the frontend will then access. 
    - Metrics from the facade.py that take a git url should go under the //GIT section in this file
    - Most of your metrics are going to belong in the //GROWTH, MATURITY AND DECLINE section. 
```javascript
    // IN THIS SECTION of augurAPI.js DEVELOPER NOTE
    if (repo.owner && repo.name) {
      // DIVERSITY AND INCLUSION
      // GROWTH, MATURITY, AND DECLINE

      // FIND THE RIGHT SECTION, like "GROWTH, MATURITY AND DECLINE" and ADD YOUR code
      Timeseries(repo, 'closedIssues', 'issues/closed')
      Timeseries(repo, 'closedIssueResolutionDuration', 'issues/time_to_close')
      Timeseries(repo, 'codeCommits', 'commits')
      // Timeseries(repo, 'codeReviews', 'code_reviews')

      // THIS IS THE NEW METRIC IN OUR EXAMPLE
      Timeseries(repo, 'codeReviewIteration', 'code_review_iteration')
```
5. 