## Background

The other day I started to notice that our Rails/PostgreSQL web app, [a cloud-based business intelligence app](https://holistics.io) is a bit sluggish. Reaching to our tracking metrics, I found out that the app's average response time is now a whopping ~200ms!

Since some of our UX interaction depends directly on the response time and to [have a smooth user interaction](https://www.nngroup.com/articles/powers-of-10-time-scales-in-ux/), they need to stay under 100ms, ideally under 50ms, I was determined to start a quest on optimizing our app.

## Where to start?

The first step in doing any kind of optimization is to find out the bottleneck. And you can only do that with visibility on the system's performance. In other words, you start by adding tracking. For each request, Rails logged the total response time, together with time spent in the controller, the view and ActiveRecord. You can build your own tracking logic by sending the logs to a central server, then parse the logs and store the result data somewhere with querying capabilities.

If you are serious about your web application and you don't have any tracking, add one as soon as possible. The cost of a tracking service like NewRelic or in our case, ScoutApp is neglible compared to the insights that they provide. We use ScoutApp since it not only provides the summary metrics but also the ability to drill down on the sluggish endpoints, not to mention the valuable feature of  n + 1 issue detection (I will come to that later).

[[ScoutApp screenshot here]]

Note that for summary metrics, don't just focus on average response time, but look into 75th and 90th percentile to have a more accurate picture of your app's performance. This is important since some very long requests or a lot of short requests may skew your perception about the app's performance.

Once you have your tracking data, focus on:

* Ones with long running time
* Ones that noticably degrades user experience
* Ones with high throughput (executes a lot of times)

In our case, we detected the main 3 problems after looking at the tracking data:

* Navigating folder hierarchy feels very sluggish
* Business object list (e.g. list of users in management page, list of reports in analytics page, etc.) takes a long time to render
* A complex query used in our job scheduling subsystem takes longer time than expected to run

## Slow to navigating folder hierarchy

In our BI app, we organize the reports into a familiar hierarchy of folders, similar to a file system or Google Drive. As we provide a pretty sophisticated permission system at each level in the hierarchy, the permission checking logic got quite complex. As more permission rules are added, browsing through the hierarchy gets slower and slower. As we did some profiling through the code, we found that some endpoint used during the browsing process sends out up to 200 database queries and 80% of the request is spent on those queries.

Our models look like this:

    class Folder < ActiveRecord::Base
      has_one :parent
    end
    
    class Report < ActiveRecord::Base
      has_one :parent, class: 'Folder'
    end

As a results, the code to get, say, the code to get the ancestors of a report looks like this:

    def ancestors(report)
	  res = []
      folder = report.parent
      while folder.present?
        res << folder
        folder = folder.parent
      end
	  res
    end

As you can see, at each level of the hierarchy, a query is sent to the database to retrieve the parent folder of the current one. Together with permission checking at each hierarchy with a bunch of similar logic, our app is sending out hundreds of queries just to show the user when she browses through a folder.

I first thought about caching the whole hierarchy in our cache (Redis) to avoid bombarding our database with such queries but the trade off is cache invalidation which is very error-prone and hard to maintain as new permission rule is added to the system.

After some more research, I found out that with PostgreSQL, there is a way to query hierarchical/graph-like data with a single query! Behold the *Recursive CTE*!

With recursive CTE, I can now retrieve the whole hierarchy with a query like so:

	with recursive tree as (
	  select R.parent_id as id, array[R.parent_id]::integer[] as path
	  from #{Folder.table_name} R
	  where id = #{folder_id}
	 
	  union
	 
	  select C.parent_id, tree.path || C.parent_id
	  from #{Folder.table_name} C
	  join tree on tree.id = C.id
	) select path from tree
	where id = 0

As I replaced the old logic with these queries, our browsing response time reduces from hundreds milliseconds to under 50ms, a huge step forwardt!
