---
layout: doc_page
---

## Theta Sketch Pig UDFs

### Instructions

* get jars
* save the following script as theta.pig
* adjust jar versions and paths if necessary
* save the below data into a file called "data.txt"
* copy data to hdfs: "hdfs dfs -copyFromLocal data.txt"
* run pig script: "pig theta.pig"

### theta.pig script: building sketches, merging sketches and getting estimates

    register sketches-core-0.5.2.jar;
    register sketches-pig-0.5.2.jar;

    define dataToSketch com.yahoo.sketches.pig.theta.DataToSketch('32');
    define unionSketch com.yahoo.sketches.pig.theta.Union('32');
    define getEstimate com.yahoo.sketches.pig.theta.Estimate();

    a = load 'data.txt' as (id, category);
    b = group a by category;
    c = foreach b generate flatten(group) as (category), flatten(dataToSketch(a.id)) as (sketch);
    -- Sketches can be stored at this point in binary format to be used later:
    -- store c into 'intermediate/$date' using BinStorage();
    -- The next two lines print the results in human readable form for the purpose of this example
    d = foreach c generate category, getEstimate(sketch);
    dump d;

    -- This can be a separate query
    -- For example, the first part can produce a daily intermediate feed and store it,
    -- and this part can load several instances of this daily intermediate feed and merge them
    -- c = load 'intermediate/$date1,intermediate/$date2' using BinStorage() as (category, sketch);
    e = group c all;
    f = foreach e generate flatten(unionSketch(c.sketch)) as (sketch);
    g = foreach f generate getEstimate(sketch);
    dump g;

### [data.txt]({{site.docs_dir}}/Theta/data.txt) (tab separated)

The example input data has 2 fields: id and category.
There are 2 categories 'a' and 'b' with 50 unique IDs in each.
Most of the IDs in these categories overlap, so that there are 60 unique IDs in total.

Results:
From 'dump d':

    (a,46.91487058420659)
    (b,46.23988568048073)

From 'dump g' (merged across categories):

    (50.415577215639736)

The expected exact result would be (60.0). The estimate has high relative error because the sketch was configured with only 32 nominal entries.

### theta_setops.pig script: set operations on sketches

    register sketches-core-0.7.0.jar;
    register sketches-pig-0.7.0.jar;

    define dataToSketch com.yahoo.sketches.pig.theta.DataToSketch('32');
    define unionSketch com.yahoo.sketches.pig.theta.Union();
    define intersect com.yahoo.sketches.pig.theta.Intersect();
    define anotb com.yahoo.sketches.pig.theta.AexcludeB();
    define estimate com.yahoo.sketches.pig.theta.Estimate();

    a = load 'setops_data.txt' as (id1, id2);
    b = group a all;
    c = foreach b generate
      flatten(dataToSketch(a.id1)) as (sketch1),
      flatten(dataToSketch(a.id2)) as (sketch2);
    d = foreach c generate
      sketch1, -- pass sketches through to have all estimates in one place 
      sketch2,
      flatten(unionSketch(TOBAG(sketch1, sketch2))) as (a_union_b),
      flatten(intersect(TOBAG(sketch1, sketch2))) as (a_intersect_b),
      flatten(anotb(sketch1, sketch2)) as (a_not_b),
      flatten(anotb(sketch2, sketch1)) as (b_not_a);
    e = foreach d generate
      estimate(sketch1),
      estimate(sketch2),
      estimate(a_union_b),
      estimate(a_intersect_b),
      estimate(a_not_b),
      estimate(b_not_a);
    dump e;

### [setops_data.txt]({{site.docs_dir}}/Theta/setops_data.txt) (tab separated)

Result:

    (10.0,12.0,18.0,4.0,6.0,8.0)
