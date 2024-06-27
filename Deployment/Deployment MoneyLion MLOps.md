So if any problem in codefresh pipeline check in the dashboard which is failing and what is failing

/run-load-test <-- to do load test

Zoom in into the part where we have the request bars number has to be the load test time (x-axis)

And then screenshot the general latency

Must do load test and put the chart latency in github PR comment

For production
1) Once code is reviewed perform the load test
2) Upload load test result
3) Get mlops approval
4) Merge code to master create the github release

2nd way is to adjust codefresh build and allow that particular dev branch and then create the release but mark it as pre-release

