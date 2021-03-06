Requires the following perl modules:
```
Scalar::Util
```

Get input files
```
wget https://s3-us-west-2.amazonaws.com/mayo-bic-tools/variant_miner/vcfs/1KG.chr22.anno.infocol.vcf.gz
wget ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20110521/README.sample_cryptic_relations
wget ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20110521/phase1_integrated_calls.20101123.ALL.panel
```
Transform the VCF sampleFormat and INFO data into JSON for uploading to the database
##### Some important notes:
* If values are strings, then they get quotes
* I skipped any data point that had a null value (`'.'`)
* The INFO field has some HTML links with some funky characters, so be careful when parsing
* I transformed the AD field from an array [in the VCF] to AD_1 and AD_2 so I could query and index it easier
* I gave the VCF a study name to merge with the sample names so I can prevent conflicts in the namespace when adding new samples
* I created a varID key by combining "chrom:pos:ref:alt" so I know what variant position I am talking about
```
bgzip -dc 1KG.chr22.anno.infocol.vcf.gz |perl scripts/VCF2Mongo.pl -VCF - -study 1000Genomes 

```

Transform the cryptic relatedness file into JSON
##### Some important notes:
* I had to skip several lines that didn't have any information

```
perl scripts/parseRelatedness.pl -i README.sample_cryptic_relations
```

The last transformation is for the population descriptions
##### Some important notes:
* This file was supposed to have 4 columns, but someone got a little liberal with the comma usage
 * replacing `ABI_SOLID,` with `ABI_SOLID` fixes the issue

```
perl scripts/parsePopulations.pl -i phase1_integrated_calls.20101123.ALL.panel
```

I now have 4 different json files to import into ArangoDB
```
#Start Mongodb
sudo mongod
#change terminal and start the import process

```


#create single entries to prime the indexes
```
for x in *json
do
 echo $x
 head -1 $x > ${x}.2
 mongoimport -d test -c ${x/.json/} ${x}.2 
 rm ${x}.2
done
```

# add indexes to collections
```

mongo --eval 'db.info.ensureIndex({ Effect_Impact: 1 })'
mongo --eval 'db.info.ensureIndex({ ExAC_Info_AF: 1 })'
mongo --eval 'db.info.ensureIndex({ SAVANT_IMPACT: 1 })'

```

#Load full collections
```
mongoimport -d test -c info info.json
mongoimport -d test -c sampleFormat sampleFormat.json
mongoimport -d test -c cryptic cryptic.json
mongoimport -d test -c populations populations.json

```

#Build the query
```
mongo
#Find the samples to keep
var getSample = function(doc) { return doc.samples; }

var excludeSamples = db.cryptic.aggregate([
    {   $match: {"Relationship": "Sibling","Population":"ASW"} },
  {
        "$group": {
      "_id": "$Population", 
      "Sample_1": { "$addToSet": "$Sample_1" }, 
      "Sample_2": { "$addToSet": "$Sample_2" }
    }
  },
  { 
    "$project": {
      "samples": { "$setUnion": [ "$Sample_1", "$Sample_2" ] }, 
      "_id": 0
    }
   }
 ]).map(getSample)[0]

```

