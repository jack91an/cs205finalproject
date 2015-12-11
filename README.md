# cs205finalproject
Final Project for CS 205

#Movie Matching System#

#####Which movie is this scene from?#####
<img src="http://2.bp.blogspot.com/-ip-SIVBkwfM/TaaJHL6VPmI/AAAAAAAAMVU/3LGPMETvc5M/s1600/the-godfather-mafia-movie.png" width=480/>


In this project, we aims to design a system that could identify a short clip from a movie and match the clip to the original movie among a large movie database.

####Method####

We researched multiple ways this matching could be done, and came across a paper by Wang et al. on the Shazam system, a music matching software that has proven to be efficient and reliable. We implemented a similar algorithm to match the soundtrack of the clip to the soundtrack of the underlying movie. In our case, movies are much longer than songs, so computation techniques that makes the matching go faster are particularly valuable.

Shazam paper: http://www.ee.columbia.edu/~dpwe/papers/Wang03-shazam.pdf

Here is a quick overview of how Shazam works:
Any soundtrack is first converted to a 2D map, where the most distinguishable sounds are marked (in the graph below). We use the amplitude of the sound to be the filter, and extract the only time and frequencies of the loudest sounds. This way, the amount of data need to be stored and computed decrease dramatically.

<img src="https://github.com/jack91an/cs205finalproject/blob/master/fp_peaks.png" width=480/>

To store the reduced data, Shazam utilized a "hash" structure. With a pre-defined "window", Shazam computes the difference between the time a sound occurs and each of the later sound that is included in the window. This sound is thereafter (not uniquely) identified as the combination of the frequencies of the two sounds and the time-offset, which we call the "hash". With storing the time that the first sound occurs as an additional variable, now we can construct a set of (hash, t1) pairs, which should be drastically different between two different songs. This set of hash pairs are the fingerprints.

<img src="https://github.com/jack91an/cs205finalproject/blob/master/fp_hash.png" width=480/>

To match a clip to a soundtrack, we would need to generate the fingerprints for botht the clip and the soundtrack. Then the hashes are compared, and due to the fact that the clip may start from the middle of the movie, there is usually a fixed difference in the t1 values of the matched hashes. We put the differences in t1 values into 200 buckets, and the most frequent t1 difference is then identified as the best estimate of when from the movie is the clip being recorded, and the count of the most frequent t1 differences is an idicator of the goddness of match. When this count is compared across all soundtracks that are compared to a clip, the soundtrack with the highest count of same t1 offsets is chosen as our system's best guess of where the clip comes from.

<img src="https://github.com/jack91an/cs205finalproject/blob/master/fp_hist.png" width=480/>

####Data####

We downloaded the 17 soundtracks from the Godfather trilogy, which contains background music across three movies with similar use of music style and instruments. The total size is 592MB and the soundtracks are about one hour long with extremely high quality .wav formats. There is no voice talking over the music, which makes it additionally difficult for our algorithm, as human voice is one of the most distinguishable sounds in our fingerprints. We believe this way we will get a baseline accuracy for our system.

Excerpts are made by functions in this notebook to directly cut a 5 second sample from each original soundtrack, and recordings are made by playing each soundtrack for 5 seconds and 10 seconds, while using another recording function in this notebook to record the clip from another computer. There are background noises for the recordings.

####Instruction####

This notebook is ready to be run through. Note that I have already put the excerpts and recordings as well as the fingerprints in the data folder, which needs to be in the same directory as the notebook. Feel free to re-run all of them, except for the recordings because it's a manual process. I have commented out the code for the recording part. I would expect to see similar results on any machines, and a full run-through should take about 30 minutes, most of which is spent on reading the original soundtracks into fingerprints (which you can skip). Key statistics will be printed out immediately following the execution.


####Performance####
<img src="https://github.com/jack91an/cs205finalproject/blob/master/results.png" width=960/>

We are glad to see that we achieved speedup in both paralellizations.

- Make fingerprints with multiprocessing
By applying multiprocessing, we did accomplish a 2x speedup with 4 cores. This is less than linear, which I think is mainly due to overhead costs. We used htop to ensure that all cores are utilized during the run:

<img src="https://github.com/jack91an/cs205finalproject/blob/master/htop.png"/>

- Match hashkeys with OpenCL
Using OpenCL, the overall time spent on searching 18 excerpts/recordings over 18 original soundtracks decreased slightly (x0.9). We thing the major costs come from arrays and variables being passed into the kernal. However, if we only time the matching process, we achieved significant speedup, at 600-800 times faster, while maintaining similar levels of accuracy. This is exciting, because as the database scale and in the real life when only 1 clip will be searched to much longer soundtracks, the matching process will take up more weight in the computation time, and the soundtracks' fingerprints won't need to be passed into repetively (which seems to be the case here). Therefore, our speedup is very scalable, and we expect the total time to decrease significantly comparing to the serial scenario when the original soundtracks grow much larger.

In terms of accuracy, both serial and parallel versions achieved a 100% accuracy when the excerpts are matched, which is expected because there is no noise nor deletion in the sound data. When matching recorded clips, both the serial and parallel version are relatively accurate, especially given the fact that there is no human voice and some of the pieces in our database are really similar. We observe that increasing the length of the excerpt did manage to improve accuracy in the parallel scenario, without incurring much extra time.


####Design####

There are two parts of the design:

1. Algorithm
Before we applied the Shazam-like algorithm, we were using a much less efficient method (byte difference) to compare two soundtracks. This method not only produces huge datafiles to be stored, but is also generally slow.
After switching to the hash structure, we were able to get more than 10x speedup, and only need to store less than 5% of the original files.

2. Parallelism
We chose to use multiprocessing to parallelize the finger creation process, because of several reasons:

- This is a one time process. Once the data is read in, it is stored and does not need reloading. Therefore, resources should be spent toward the matching part, and multiprocessing is a simple and effective (2x speedup) solution.

- Several python libraries were used in the process, which makes it hard to get into the package and customize the code (like for Cython).

On the other hand, we decided to explore OpenCL as the tool to parallelize the matching process, as utilizing GPU would speed up the process significantly (700x). Here are some of the details of our OpenCL design:

- Instead of passing the list of (hash, [t1, t2, ...]) tuples into the matching function, we needed to pass fixed-length items and decided to instead pass [(hash, t1), (hash, t2), ...] into the kernal. This step was done outside of the matching function, and takes roughly the same amount of time comparing to making fingerpy rints that are fed into the serial code.

- Another challenge was that in parallel we need to build the "bucket" list, which stores the number of t1 offsets that fall into a particular range. Due to the fixed-length constrained, we split the whole soundtrack time into 200 equally sized buckets, and use a counter to keep track of corresponding number of matches.

####Next Steps####

There are a few simple functionalities we would like to add to the code but didn't have time to get to:

- The ability to tell what time the excerpt/clip starts in the original movie. This would be a very helpful function, which generally not valuable in song matching.

- We also want to test our methodology on clips where there is no background music but only human speaking, as this is an often scenario for clips people are searching for.

- We would like to explore ways that would decrease the overhead when passing variables into OpenCL. This is currently the bottle neck for better results under the same settings

####Reflection####

I really enjoyed this project because of the iterative process that we explored through, from deciding on the serial algorithms to deciding on the parallelism methodologies. Here we are really thankful for Ray, who spent a lot of time sitting with us challeging our thinking and pushing us to work on interesting tasks.

In this project, the initial designing of the system took a lot of time, which left the room for parallelization short of ideal. This is personally my first software project, and I have learned a lot about design thinking and basic computer engineering knowlegde from the professors, TFs and my teammate. What applies to the future will be the mindset of planning on a high level and setting priorities so the time would be spent on the most significant improvement.
