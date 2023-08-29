# Private Ad Measurement

We propose a system for privacy-preserving ad attribution. We link information between impressions and conversions in the browser and encode them as histograms. Attributions are transmitted using the Prio ([<u>Prio | Stanford Applied Crypto Group</u>](https://crypto.stanford.edu/prio/)) framework to create aggregate reports of those histograms for advertisers. The proposal is intended to handle arbitrary attribution logic and provide Differentially Private (DP) ([<u>Differential Privacy</u>](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/dwork.pdf), [<u>The Algorithmic Foundations
of Differential Privacy</u>](https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf)) guarantees on final aggregates.   

## Outline

Here’s a quick overview of what this document covers before we go into the details.  

First we describe how to store information from ads on a device. The storage is local and written at ad impression time. Much like a browsing history, users should be able to manage this storage but it should not be readable or even detectable by websites. The information stored can be arbitrarily fine-grained but will not leave the device in that level of detail. Instead the on-device storage functions as a distributed ad impression database that will enable specific queries described later.  

Second we describe what conversion information is registered on-device and how that information is used to derive a histogram. This histogram attributes the conversion to matching ad impressions in the user's storage.  

Third we describe how the histogram is sent to a Prio aggregation system that adds central DP noise to its aggregates in order to provide privacy guarantees. The management of a privacy budget across both the on-device data store and the aggregation system is discussed.  

Fourth we suggest extensions to regimes with very large numbers of unique advertisements that fit within the same framework.  

Finally, we describe how publishers can use this system to collect information. In order to accomplish this, an additional fixed time delay from publication is introduced. This prevents tying information to a user or device through mere time matching of impression and conversion events.  

## **Impression Data Recorded On Device**

This system would require an advertisement to register the following pieces of information with the browser at the time of impression:  

- Identity of the ad measurement system. Often run by an adtech vendor and responsible for reporting the attribution of an advertisement.
- Identity of the advertiser.
- The ad ID. This could be a coarse identifier like a single ad campaign, one of two IDs to perform A/B comparisons, or more a granular identifier. Eventually these IDs will be aggregated across users and the aggregates will have differentially private noise added. The number of attributions will determine how much signal can be derived and can inform which granularity to pick for the identifier.
- Maximum times the ad impression can be attributed. An advertiser may want to convert repeatedly for a single ad impression. Doing so changes the amount of differentially private noise that needs to be added to an aggregate. Coordination between how many times an advertiser queries an impression and this maximum number will need to be done in the external system managing advertisement and conversion logic (e.g. the ad-tech defining ad IDs).
- For how long this ad impression should be stored.
- Type of user engagement. Click, view, etc.

The device records meta data along with the ad impression. For example:  

- Time.
- Duration of ad interaction.
- Publisher website.

The above data is stored in the browser and cannot be queried or detected by any website. The meta data can be used to select the most relevant impression at the time of conversion as described below.  

## **Conversion Data Recorded On Device**

At the time of conversion, an advertiser wishes to attribute credit for the conversion to a user’s ad impressions. In this system, the logic for attribution is delivered to the device. For each conversion event, the advertiser registers the following information on the device:  

- Identity of the ad measurement system (potentially run by ad-tech) responsible for reporting the attribution of an advertisement.
- Identity of the advertiser interested in attributing this conversion to ad impressions.
- An identifier of the conversion event that is shared across all users converting (conversion ID). Once again this could be coarse or granular depending on the number of users participating.
- The value of the conversion.
- The list of all ad IDs that might have contributed to this conversion.
- The mapping from each ad ID to a bin in a histogram that will ultimately report the attributed value for that bin. In the simplest model, each ad ID would have its own bin and so the bin would contain the value attributed to a particular impression.
- Maximum number of times any impression involved in this conversion could be used for attribution. Allowing an ad impression to be queried (converted for) multiple times enables measurements of user journeys. However, performing multiple queries requires more DP noise to be added to the resulting aggregate for the same privacy guarantees. This maximum number determines the amount of noise to be injected for the conversion aggregate. If this number is less than the number of times a particular advertisement might be used, there would be a mismatch in privacy guarantees and a device would not participate. The ad-tech needs to enforce matches between these limits between the impression and conversion ecosystem it defines for ad impressions to be included in the attribution.
- Attribution logic for this particular conversion. For example first click, last click, all equal, time-weighted. This logic can use the contextual information recorded for each ad. The aggregation can be specified in a domain-specific language or e.g. as an enumerated list of attribution schemes.

## **Joining Impressions and Conversions**

This proposal envisions a Prio type aggregate using a histogram to encode the information for each conversion. The following procedure defines creation of this histogram:  

- For each conversion ID:
  
  - Select all ad impressions in the local data store that are in the list of ad IDs considered by the conversion. If a particular impression has been used for attribution its maximum number of allowed references, do not include it.
  
  - Using the attribution logic of the conversion, calculate a fraction of attribution per ad ID.

- Create a payload histogram by:
  
  - Creating a histogram with the number of bins specified in the conversion ID and filling each bin with the corresponding fraction of the conversion value specified by the conversion logic. Bins that contain multiple ad IDs would contain the sum of the attribution per ad ID. This can be calculated as value from conversion multiplied by fraction of attribution. In the event that a conversion does not match any impressions, a default bin would be filled. Logic that cannot fully attribute credit to impressions could put any remaining value in the default bin.
  
  - Optionally, appending to this histogram number of bins specified in the conversion ID and filling each with the corresponding fraction of the conversion value squared. This allows not just the average conversion value per bin to be calculated, but also the variance of the conversion value per bin.

- For each ad ID considered:
  
  - Increment the number of times the associated impression has been queried.
  
  - Store the payload keyed by conversion ID.
  
  - Store the results on the device so that a log of all results can be made available to the user for transparency reasons.

## **Reporting to Prio Aggregators**

Prio creates a system to report cryptographic shares of histograms to multiple parties and sum them. The system has guarantees detailed in this document: [<u>Prio: Private, Robust, and Scalable Computation of Aggregate Statistics</u>](https://crypto.stanford.edu/prio/paper.pdf).  

Every device contributes cryptographic shares identified by the conversion ID to the Prio aggregation system. These shares are created from the payload defined in the “Joining Impression and Conversions” section. The aggregation is done by conversion ID so that every conversion ID is summed across all users who have converted in this particular way. The DP noise would be determined by the maximum number times any ad ID might be used for this conversion. This allows the browser to manage the per-user budget for a particular query and the Prio aggregation to manage the per-conversion ID aggregate addition of DP noise.  

The level of noise would be specified by the ad attribution measurement system and a browser would support those that have an appropriate noise addition. The in-browser logic could also include a total measure of data sent and manage a privacy budget. This could be defined hierarchically so that each website (e.g. effective top level domain (eTLD) plus one) would have a per-device per-day budget and the browser would maintain a global budget so that a particular user would not accidentally release too much information. This could be made more nuanced by adjusting the time period (e.g. defining a session as the time period over which to maintain a budget) or the levels of hierarchy (e.g. eTLD plus two, eTLD plus one, global). The addition of complexity reduces the explainability and so should be carefully considered.

## **Handling Large Numbers of Connections Between Impressions and C**onversions

As defined, the number of conversion IDs and ad IDs considered in this system can be unbounded. However, the system so far supports a finite, somewhat small number of ad IDs associated with each conversion ID. In order to support much larger numbers of ad IDs per conversion, we consider two ways of addressing many ad IDs associated with a particular conversion ID.  

The implementation uses standard probabalistic data structures ([Category:Probabilistic data structures - Wikipedia](https://en.wikipedia.org/wiki/Category:Probabilistic_data_structures)) — in particular Bloom Filters ([<u>Bloom filter - Wikipedia</u>](https://en.wikipedia.org/wiki/Bloom_filter)), Count-Min Sketch ([<u>Count–min sketch - Wikipedia</u>](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch)), and Count Sketch ([<u>Count sketch - Wikipedia</u>](https://en.wikipedia.org/wiki/Count_sketch)).  

### **Many Ad IDs Mapped Per Bin**

We first consider the scenario where an advertiser has a very large number of ad IDs and wishes to group them together into a small number of bins. This could be, e.g. when an advertiser creates a very diverse set of ways to specify advertisements across their population (creating a unique ad ID based on hour of the day, user cohort, geographic information, publisher, or other data available to the advertiser at the time of impression) but wants to group many of these together into bins on the device. Above we suggest that this be done with a map from ad ID to bin number but in this scenario the number of potential ad IDs would be too large to deliver to the device.  

To address this, each bin can be associated with a Bloom Filter. This approximate set would be used to sort ads into bins using logic known to the advertiser. Each possible ad ID would be inserted into the Bloom Filter associated with the correct histogram bin. This enables a large set of ads to be associated with a particular bin without requiring an exhaustive list to be sent to the device. This mapping is then treated identically to the lower cardinality bins described previously.  

### **Unbounded But Noisy Lookups of Ad IDs**

We now consider a scenario where the set of ad IDs associated with a conversion ID may be unknown at the time of conversion and can be potentially unbounded. The list of all ad IDs is required to be known at the time of measurement.  

As a concrete example of where this might be useful, consider a situation where an advertisement is displayed on many websites that may not be known beforehand. The device can record the particular site where an ad impression occurs as the ad ID stored in the in-browser impression database. This data will be used later at the time of conversion, should one occur. This impression data must also be stored by the advertiser so that all possible ad IDs are known to the advertiser when advertising effectiveness is measured. In other words, the impression stream should be stored by the advertiser.  

We propose using tools designed for the Heavy Hitters problem ([Misra, J.](https://en.wikipedia.org/wiki/Jayadev_Misra); [Gries, David](https://en.wikipedia.org/wiki/David_Gries) (1982). "Finding repeated elements". *Science of Computer Programming*. **2** (2): 143–152. [doi](https://en.wikipedia.org/wiki/Doi_(identifier)):[10.1016/0167-6423(82)90012-0](https://doi.org/10.1016%2F0167-6423%2882%2990012-0). [hdl](https://en.wikipedia.org/wiki/Hdl_(identifier)):[1813/6345](https://hdl.handle.net/1813%2F6345).). The Count-Min Sketch ([<u>Count–min sketch - Wikipedia</u>](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch)) and Count Sketch ([<u>Count sketch - Wikipedia</u>](https://en.wikipedia.org/wiki/Count_sketch)) algorithms create finite memory approximate lookup table of frequency for a given input. These have two properties that make them appropriate for this system.  

- They are serialized as a histogram and so can be reported to a Prio aggregation system.
- The sketch for a set of inputs is identical to the sum of sketches made from each input.

To finish our example, each device undergoing a conversion inserts the relevant ad IDs (those that are from the advertiser where the device is converting) into a local sketch. This sketch is sent to a Prio system to be combined with all the other sketches from this conversion ID. The sketch over the entire population is reported to the advertiser. The advertiser has the impression data from their advertising campaigns and so knows the list of ad IDs (in this example, websites where an impression occurred) and how many impressions per ad ID. The number of conversions per ad ID can be looked up in the sketch giving a ratio of conversions per impression. If one were interested in conversions of different values, the number of times a particular ad ID is inserted into the sketch can also be increased on the device so that a total value per ad ID could be measured.  

Just as in the version of this system without approximate data structures, the advertisements considered for attribution can be selected using metadata from the in-browser impression database with logic delivered at the time of conversion. So, e.g. only the most recent ad impression could be added to the local sketch rather than all ad impressions.  

## Publisher Reports

The system can report to publishers as well as advertisers. This does introduce a new consideration. When one reports to an advertiser at the time of conversion, no additional information is transmitted — the conversion timestamp is already known to that advertiser. Reporting to the publisher should not reveal information that could be derived from knowing the impression time and conversion time. If a publisher Prio share was sent off device at the time of conversion, this could be closely tied to the time of impression and so reveal information to a publisher linking a particular user to a particular share — and reveal that a conversion occurred. To deal with this extra wrinkle, we introduce a time delay between ad-impression and reporting set by the publisher at the time of impression. With this change, the timing of reporting information to a publisher is not effected by the timing (or even existence) of a conversion.  

To use the system for publisher data, the publisher registers its own copy of the advertisement information at the time of impression:  

- Identity of the ad measurement system responsible for reporting the attribution of an advertisement to the publisher.
- Identity of the publisher serving this ad.
- A publisher-specific identifier of the advertisement (publisher ad ID) that is shared across all users encountering the ad. Eventually these results will be aggregated across users and have differentially private noise added to them so the number of users with impressions will determine how much signal can be derived and the appropriate granularity of the identifier.
- Length of time this ad impression should be stored on the user’s device before reporting impression information to the publisher.

Since each impression is attributed a conversion value on-device, the in-browser database contains those values for all publisher ad IDs (or an empty value for impressions that did not result in a conversion). When the publishers’ specified in-browser measurement time expires, the device reports the conversion value (or zero) to the publisher’s advertising measurement system. Reports for each publisher ad ID can be aggregated and augmented with DP noise in the same way an advertiser’s information is.
