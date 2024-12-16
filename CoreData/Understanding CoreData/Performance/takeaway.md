Summary
In this chapter, we introduced a mental performance model of Core Data by splitting the stack into three tiers: the context tier, the coordinator tier, and the SQLite tier. We then discussed how fetch requests are generally expensive, and how to avoid them in many situations. Next, we took a look at how to make sure fetch requests perform well when we need them. Most of this chapter is about trying to stay within the context tier whenever possible, and how to ensure that SQLite can do its work as efficiently as possible whenever we actually need to drop down to the SQLite tier.