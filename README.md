# Swift@IBM Evolution Process

This repository tracks the ongoing evolution of the frameworks and capabilities provided by the Swift@IBM team. It contains:
* **Manifestos:** describing the proposed roadmaps and goals for major projects, eg. Kitura.
* **Proposals:** describing the design proposals for new features or changes to existing capabilities.
* **Status:** describing the status of the current proposals against that major project area
* **Releases:** describing which past proposals were delivered in which release of a functional area

These are provided under subdirectories for each of the major project areas, currently:
* [Kitura](https://github.com/IBM-Swift/evolution/tree/master/Kitura): for the Kitura framework itself
* [Other](https://github.com/IBM-Swift/evolution/tree/master/Other): for all other projects  

This will shortly be extended to provide specific areas for other major projects including: Monitoring, Swift Server Generator, etc
 
## Evolution Process

The Swift@IBM Evolution process is modeled on the [Swift Evolution process](https://github.com/apple/swift-evolution/blob/master/process.md), but covers multiple functional areas, including all of the projects in the Swift@IBM organisation in GitHub. 

### Goals
The Swift@IBM Evolution process aims to more publicly document the requirements, goals, architecture and design of capabilities and features being added to the Swift@IBM projects, and to leverage the feedback, ideas and experiences of the Swift community in order to provide capabilities that the community wants, in the way that they want them.

### Participation
Everyone is welcome to propose, discuss, and review ideas to improve the Swift@IBM projects. Unlike the Swift Evolution process, there is no mailing list for discussion: discussions and reviews we be carried out using Slack and via GitHub pull requests and issues.

### How to propose a change
* Consider any documented goals in that functional area:  
Each of the major functional areas will eventually have a published manifesto describing the function areas being considered for the next release(s). Proposals that are within the scope of the relevant manifesto are more likely to be considered. It is recommended that proposals outside of the scope of the relevant manifesto are well socialised in order to get initial feedback.
* Socialize the idea:  
Before creating a proposal, the idea should be discussed via Slack, stating the use case, and the problems it solves, along with an idea of what the solution might look like in order to gain interest from the community.
* Develop the proposal:  
Proposals should be developed using the proposal [template](https://github.com/IBM-Swift/evolution/blob/master/template.md). Prototyping an implementation and its uses along with the proposal is encouraged, because it helps ensure both technical feasibility of the proposal as well as validating that the proposal solves the problems it is meant to solve.
* Request a review:  
Submitting the proposal via a pull request to the [ibm-swift/evolution](https://github.com/IBM-Swift/evolution) repository starts a review of the proposal submission, ensuring that it is sufficiently detailed and clear. Once accepted as a proposal, a proposal number and a Swift@IBM team member will be assigned to the proposal. A GitHub issue will then be opened in order to review the proposal itself.
* Address feedback:  
Feedback provided during the review should be responded to and addressed.

### Review process
Once the proposal is accepted and a GitHub issue created, the review manager will work with the proposal authors generate and address feedback. The review period will typically be one week, but may run longer if there is still ongoing discussion that needs to be addressed.

After the review has completed, the review manager is responsible for determining consensus and reporting the decision to the proposal authors. The review manager will then update the proposal's state in the [ibm-swift/evolution](https://github.com/IBM-Swift/evolution) repository to reflect that decision.

### Proposal states
A given proposal can be in one of several states:
* Under review  
* Returned for revision  
* Withdrawn  
* Deferred  
* Accepted  
* Accepted with revisions  
* Rejected  
* Implemented (release version)  

### Implementation of proposals
Once a proposal has been accepted, an issue will be created in the relevant project to implement the proposal. Participation from the community to implement proposals is encouraged. It is recommended that you contact any people assigned to the issue in order to collaborate with them, or discuss with review manager how to get started implementing the proposal for any unassigned proposals.