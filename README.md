# Social Media Aggregator Design Doc

* [One Month](/MONTH_ONE.md)
* [One Year](/TWELVE_MONTH.md)

## Included in this Doc

This design doc focuses on overall systems topology with thorough explanations for my design decisions.
My main focus in this design was to create one service design that will easily scale with the a growing
feature set and user base. By starting out with a simple codebase with stricter separation of concerns
we can easily migrate pieces out of that codebase without impacting other dependent modules/services.

I also included proposals for scaling, observability, and monitoring in the early proposal that already would
ideally meet the requirements for the later iterations.

## Not included in this Doc

Proposals for deploying this architecture in a multi-region or edge enabled architecture. In order to
achieve the latency requirements at the one year mark, moving the API closer to where the user is might
be a vital step. This process is a bit outside of my experience but would be an amazing challenge to
overcome and give me a set of tools I have desired for a long time.
