---
layout: post
title: ""
date: 2020-07-29
author: Stephanie Zendejo
---

# My Approach To The Changelog Problem
> Changelog problem? An introduction to MABE and the problem can be found [here!](https://szendejo.github.io/waves/blog/Team-MABE.html)  

We can think of the parent genome as a std::vector of sites. Each site in the parent genome contains a numeric value that is represented as a byte in memory. The position of the site in the parent genome is the index.  

|   |   |   |   |   |   |    |   |
| - | - | - | - | - | - | -  | - |
| 2 | 3 | 4 | 1 | 2 | 0 | 4  | 1 |

<!-- [Parent Genome in all its glory](https://i.imgur.com/mekOG1s.png) --->

A changelog is represented as a std::map<size_t, Site>. Size_t is the index of the site mutated, and Site is the struct to contain the mutated site's information. The Site struct identifies what type of mutation has been applied to the site, and what the new value is (if applicable).  
```c++
struct Site {
	size_t insertOffset;  	  // insert mutation at site
	size_t removeOffset;   	  // remove mutation at site
	std::byte value;       	  // mutated number value
};	
std::map<size_t, Site> changelog; // key is index of site in the parent genome
                                  // value is Site structure
std::vector<std::byte> sites;     // parent genome
```
### Mutation Signatures
Here's a basic idea of what each of the functions do. I followed the mantra of _when in doubt, shift it out_.  
**Overwrite**  
  * Loops through segment vector
  * Adds overwrite mutations to the changelog  
  
Insert Overwrite Mutation Gif here  
<!-- ![OverWrite Example](https://i.imgur.com/wu7gBxK.gif) -->

**Insert**  
  * Shift sites in the changelog to the right by size of the segment vector
  * Loops through segment vector
  * Adds insert mutations to the changelog  
  
Insert Insert Mutation Gif here  
<!-- ![Insert Example](https://i.imgur.com/0rZ4Bai.gif) -->

**Remove**  
  * Removes sites in the changelog if they exist
     * Takes into account if sites removed in the changelog had insert or remove mutations
  * Shift sites in the changelog to the left
  * Adds remove mutation to the changelog  
  
Insert Remove Mutation Gif here   
<!-- ![Remove Example](https://i.imgur.com/tus7plB.gif) -->

 
The overwrite and insert signatures contain a segment vector as an argument. The segment vector is a collection of mutations that will be added to the changelog starting at the given index. 
> _insert(6, {44, 55, 66}) is the equivalent of inserting the following mutations to the changelog:_   
> _site at index 6 with a value of 44_  
> _site at index 7 with a value of 55_  
> _site at index 8 with a value of 66_  
```c++
virtual void overwrite(size_t index, const std::vector<std::byte>& segment); 
		// Ex. Starting at index 5, overwrite 3 sites with the values 11, 22, 33

virtual void insert(size_t index, const std::vector<std::byte>& segment);    
		// Ex. Starting at index 6, insert 3 new sites with the values 44, 55, 66

virtual void remove(size_t index, size_t segmentSize) override; 	     
		// Ex. At index 7, remove 3 sites
```

### Adding Entries In The Changelog
Let's apply some basic mutations to a parent genome.  
Insert Parent Genome picture here

1. Overwrite mutation to site at index 2. The overwritten sites will have values of 11, 22, 33.  

| Key | Site Value | Remove Offset  | Insert Offset |  
| --- |:----------:|:--------------:| -------------:|  
|  2  |     11     |       0        |       0       |    
|  3  |     22     |       0        |       0       |    
|  4  |     33     |       0        |       0       |  

> _Segment vector contains 3 site values. Starting at index 2, each site value will be either added in the Changelog or edited if it exists. Index key 2 does not exist, so it's inserted in the Changelog. Site Value is set to 44. Since this is an overwrite mutation, the size of the offspring genome is not affected. Remove Offset and Insert offset are both set to zero. The steps are repeated for each site value in the segment vector._    

2. Insert mutation to site at index 1. The inserted site will have values of 44, 55, 66.  

| Key | Site Value | Remove Offset  | Insert Offset |   
| --- |:----------:|:--------------:| -------------:|  
|  1  |     44     |       0        |       1       |    
|  2  |     55     |       0        |       1       |    
|  3  |     66     |       0        |       1       |   
|  5  |     11     |       0        |       0       |    
|  6  |     22     |       0        |       0       |    
|  7  |     33     |       0        |       0       |  

> _Shift all sites in the Changelog to the right by 3, the number of sites inserted. Site at key 2 becomes site at key 5. Add entries to the Changelog map starting at index 1 with their values. Because these are insert mutations, Insert Offset is set to 1. Offspring genome size increases by 3._  
 
3. Remove mutation to site at index 6. Remove 3 sites.   

| Key | Site Value | Remove Offset  | Insert Offset |   
| --- |:----------:|:--------------:| -------------:|  
|  1  |     44     |       0        |       1       |    
|  2  |     55     |       0        |       1       |    
|  3  |     66     |       0        |       1       |   
|  5  |     11     |       0        |       0       |    
|  6  |      0     |       3        |       0       |   

> _Starting at index 6, sites 6 7 and 8 will be removed. Sites 6 and 7 exist in the Changelog. They are not insert or remove mutations so they can be easily removed. A new entry is added at site 6, with a Remove Offset of 3._

Great! All mutations have been recorded. Much like this rendition of Celine Dion's _My Heart Will Go On_,  
Insert youtube video here  
<!--- https://www.youtube.com/watch?v=X2WH8mHJnhM -->
this genome ~~heart~~ will go on to the next generation. We're going to use the changelog on the parent genome to generate the offspring genome. 

### Generating The Offspring Genome  
A vector named modifiedSites contains the offspring genome. Each position in modifiedSites will be populated from either the changelog if an entry exists, or from the parent genome. 
Talk about the changelog insert and remove offsets and how that affects the index in the parent genome  
```c++
void StephanieGenome::generateNewGenome() {
	modifiedSites.resize(genomeSize);
	int offset = 0;

	// Create an offspring genome
	for (int i = 0; i < genomeSize; i++) {

		// Index does not exist in the changelog
		if (!changelog.count(i)) {
			modifiedSites[i] = sites[i + offset];
		}

		// Index exists in changelog and site is a overwrite mutation
		else if (changelog.count(i) && (changelog[i].removeOffset == 0 && changelog[i].insertOffset == 0)) {
			modifiedSites[i] = changelog[i].value;
		}

		// Index exists in changelog and site is an insert mutation
		else if (changelog.count(i) && changelog[i].insertOffset > 0) {
			modifiedSites[i] = changelog[i].value;
			offset -= 1;
		}

		// Index exists in changelog and site is a remove mutation
		else if (changelog.count(i) && changelog[i].removeOffset > 0) {
			offset += (int)changelog[i].removeOffset;
			modifiedSites[i] = sites[i + offset];
		}
	}
}
```  
Insert gif about reading changelog and the index in parent genome and offset genome

# Time vs. Memory  
## Benchmarking  
Talk about benchmarks  
Insert Graphs
## Optimization  
Talk about improving the code itself

## Verdict  
Talk about how the Test Genome is ultimately faster ;__;

# Lessons Learned  
Talk about lessons learned  

# Acknowledgements
**Mentors:** Clifford Bohm, Jory Schossau, Jose Hernandez  
**Team Members:** Jamell Dacon, Tetiana Dadakova, Victoria Cao, Uma Sethuraman  

This work is supported through Active LENS: Learning Evolution and the Nature of Science using Evolution in Action (NSF IUSE #1432563). Any opinions, findings, and conclusions or recommendations expressed in this material are those of the author(s) and do not necessarily reflect the views of the National Science Foundation.

# Resources

