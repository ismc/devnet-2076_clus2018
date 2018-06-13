#### Show Initial State
1. *Steven*: Show on the production network that spoke-to-spoke traffic passes through the hub

#### Unit Testing
1. *Chris*: Change `no ip nhrp shortcut` to `ip nhrp shortcut`
  1. `cd brkarc-2023_clus2018/roles/network-dmvpn`
  1. `git checkout devel`
  1. `vi templates/ios-dmvpn.j2` (make change)
  1. `git status`
  1. `git commit -am "Enabled spoke-to-spoke"`
  1. `git push origin devel`
1. *Chris*: Go to `https://github.com/ismc/ansible-network-dmvpn` and create a PR
1. *Observe the test start running in the PR*
1. Observe testing start in Jenkins: http://jenkins1.sjc.ismc.io:8080/blue/organizations/jenkins/ansible-network-dmvpn/activity
1. *Observe the PR in Webex teams*
1. *Observe the successful test in Webex Teams*
1. *Chris*: Message Steven `please review/approve my PR`
1. *Steven*: Go to `https://github.com/ismc/ansible-network-dmvpn/pulls`
  1. When tests pass, review and merge

#### Integration Testing
1. *Chris*: Update the main repository with the
  1. `cd brkarc-2023_clus2018`
  1. `git checkout devel`
  1. `git submodule update --remote roles/network-dmvpn`
  1. `git status`
  1. `git commit -am "Enabled spoke-to-spoke"`
  1. `git push origin devel`
1. *Chris*: Go to `https://github.com/ismc/brkarc-2023_clus2018` and create a PR
1. Observe the PR in Webex Teams
1. *Steven*: Go to `https://github.com/ismc/brkarc-2023_clus2018/pulls`
  1. Type `test it` in the comment
  1. *Observe the test start running in the PR*
  1. Observe testing starting in Jenkins: http://jenkins1.sjc.ismc.io:8080/blue/organizations/jenkins/brkarc-2023_clus2018/activity
  1. When tests pass, review and merge

#### Push Change out to Production Network
1. *Steven*: Go to https://tower1.sjc.ismc.io/
  1. Run Provision Site Routers workflow

#### Show Final State (i.e. New artifact pushed to production)
1. *Steven*: Show on the production network that spoke-to-spoke traffic goes direct
