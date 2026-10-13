# wiz_image_update


Wiz Broker & Outpost Lite: Side-by-Side
V alidation for Upgrades
Oct 13, 2
Summary
We test every new Wiz Broker and Outpost Lite version in a separate green environment
before it touches the live blue environment. Green runs the new Wiz image and scans a
few GitLab repos. Once it passes, we upgrade blue to the same version.
The key point: we never switch traffic to green. Blue stays the live environment the
whole time. Green exists only to prove the new version works in our real network.
This supports WIZDEFEND-248 (frequent patching with less manual work) and
WIZDEFEND-508 (BAU upgrade process).
Note on naming: this is not classic blue/green, because there is no cutover. A more
accurate name is side-by-side validation. We keep the blue and green labels because they
are easy to follow, but reviewers should not expect a traffic switch.
The two environments
Blue and green run side by side in the same EKS cluster, in separate namespaces. Each
has its own Wiz Broker, Outpost Lite pods and Wiz connector.


Blue handles all real scanning and reports to the prod Wiz project. Green only scans a few
test repos, through its own connector, into a separate Wiz project so test results never
appear in prod dashboards.
Blue Green
Purpose Live scanning T esting the new version
Wiz version Current New
Repos scanned All A few test repos
Wiz connector and project Prod Separate test ones
Gets live traffic Y es Never
Lifetime Permanent Only during testing
Upgrade flow
Every new version goes through green first. Blue is only upgraded after green passes and
the change is approved.




1. New version found. The daily pipeline detects a new Wiz Broker or Outpost Lite
release and copies the image into Nexus and ECR.
Page 2 of 5
Wiz Broker & Outpost Lite: Side-by-Side Validation for Upgrades
2. Deploy to green. The new version is installed in the green namespace with its own
Helm release. Blue is not touched.
3. Scan test repos. Green scans a few GitLab test repos and reports to the separate Wiz
project.
4. Check results. If any check fails, blue stays on the current version and we raise the
issue with Wiz or wait for the next release.
5. Change approval. The change request goes in with the green test results as evidence.
6. Upgrade blue. After approval, blue is upgraded in place to the same version that
passed in green.
7. Remove green. Green is scaled down or deleted, or kept ready for the next upgrade.
What we test in green
Green passes only if every check below succeeds. The results are attached to the change
request.
Check How we confirm it
Pods are healthy Connector is up Scans work No errors Stable over time Broker and Outpost Lite pods are Running and use
the new image tag
The test connector shows Connected in Wiz
(checked through the Wiz API)
Scans of the test repos complete and results
appear in the test Wiz project
Pod logs show no errors
Green keeps passing through a short soak period
(length to be agreed)
Upgrading blue and rollback
Blue is upgraded in place with helm upgrade --atomic, using the exact version that
passed in green. If the upgrade fails or the pods do not become healthy, Helm rolls blue
back to the previous version automatically.
If a problem shows up later, we roll back with helm rollback, or by reverting the version
change in Git so the pipeline redeploys the previous version. After any rollback, we run
the same checks again and record the reason in the Jira ticket and change request.
Page 3 of 5
Rollback only works if the previous image is still available, so we keep at least the last two
versions in Nexus and ECR.
The Broker runs as a single pod, so the blue upgrade causes a short connectivity gap.
Scans running at that moment may need to retry, so the upgrade should be scheduled at a
quiet time.
How this differs from classic blue/green
The only real difference is the last step: classic blue/green switches traffic to green, while
we upgrade blue instead.
We chose this because Broker and Outpost Lite receive no inbound traffic. They connect
out to Wiz, so there is no load balancer or service to switch. A classic cutover would mean
moving Wiz connectors between Brokers, which adds risk and manual work for little
benefit.
Assumptions and open questions
Assumptions:
Green runs in the prod EKS cluster, in its own namespace next to blue.
After green passes, blue is upgraded in place, and green is not promoted to live.
Wiz allows a second Broker and connector to be registered for green.
Open questions:
Classic blue/green Our approach
What green is for Becomes the new live
environment
T esting only
Traffic switch Y es, from blue to green No, never
How blue gets the new
version
It doesn't; blue is retired Upgraded in place after green
passes
Rollback Switch traffic back to blue Helm rollback on blue
Which GitLab repos will green scan, and who owns them?
How long should the soak period in green be?
Should green be deleted after each upgrade or kept running at zero replicas?
Does green need a separate change request, or is it covered by the blue upgrade
change?
Wiz Broker & Outpost Lite: Side-by-Side Validation for Upgrades
Page 4 of 5
Wiz Broker & Outpost Lite: Side-by-Side Validation for Upgrades
Can blue be upgraded more often than the current 2-month cadence once green
testing is in place?