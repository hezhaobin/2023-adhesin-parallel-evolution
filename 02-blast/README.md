---
title: Use BLAST to identify homologs for XP_028889033.1
author: Bin He
date: 2020-07-01
---

# Goal

- Repeat the blast step to clean up the homologs list.
    some species were missing while others, like _C. albicans_, had more than one strain represented in Lindsey's version.

# Notes
## 2020-07-01 [HB] Repeat BLAST to identify XP_028889033 homologs
### FungiDB
Used the beta version of the new site on 2020-07-01

- Used first 560 aa of XP_028889033 as query, e-value cutoff set to 1e-5, low complexity on, and limit the organisms to the CUG clade, _S. cerevisiae_, _C. glabrata_ and _S. pombe_
- Downloaded 95 sequences along with a table with meta information.
- After examining the meta data, I noticed that some sequences are much shorter than others. I then plotted protein length as a function of e-value, both in log scales, and it became apparent that those sequences below 500 amino acids are the ones with the lowest e-values. I thus removed them by printing a list of gene IDs, and used the `extract_fasta.py` program to output the filtered list.
    <img src="img/20200701-XP_028889033-fungidb-hits-e-value-by-length.png" width="480">

### Retrieve ref_protein ID for FungiDB hits
```bash
$ blastp -db refseq_protein -query XP_028889033_homologs_fungidb_use.fasta -outfmt "6 qseqid sseqid qlen slen pident mismatch score bitscore evalue" -max_target_seqs 1 -remote -out XP_028889033_fungidb_refprot_id.txt
```
However, for some reason this command didn't work (2020-07-04: I think it actually just takes a long time. Instead of returning an ID for later retrieval of the results, it appears that the user actually have to wait until the search finishes). Instead, I submitted the XP_028889033_homologs_fungidb_use.fasta to NCBI blastp with the same parameters. The result is registered with RID: G2HK3CWK014. I then downloaded the results with a local command
```bash
$ blast_formatter -rid G2HK3CWK014 -out fungidb_N560_blast_refseq_protein.txt -outfmt "6 qseqid sseqid qlen slen pident mismatch score bitscore evalue" -max_target_seqs 1
$ cut -f1 fungidb_N560_blast_refseq_protein.txt | sort | uniq | wc -l
# 70, correct
```


### NCBI blast
`blastp` with the first 560 aa of XP_028889033, e-value cutoff 1e-5, low complexity sequences masked (there is now an option to only mask the low complexity region when generating the seed, but leave them be during the extension).
- I also tried `Delta-blastp`, which first searches against the conserved domain database and then gather sequences with that domain. This resulted in way too many hits. Didn't pursue further.
- For the `blastp` results, I further required the query coverage to be greater than 50%, which yielded 144 sequences. This cutoff was chosen subjectively as sequences with lower than 50% coverage appear uninteresting (in species that are not what I'm interested in).
- I further excluded 6 species from consideration. These are "Metschnikowia bicuspidata var. bicuspidata NRRL YB-4993 (taxid:869754), Debaryomyces fabryi (taxid:58627), Suhomyces tanzawaensis NRRL Y-17324 (taxid:984487), Candida orthopsilosis Co 90-125 (taxid:1136231), Kazachstania (taxid:71245), Naumovozyma dairenensis CBS 421 (taxid:1071378), Meyerozyma guilliermondii (taxid:4929), Yamadazyma tenuis ATCC 10573 (taxid:590646)"
- The resulting taxonomy is shown ![here](img/20200704-ncbi-blastp-XP_028889033-taxonmy-distribution.png)

### Merge the two datasets
1. To merge the two datasets, I decide to blast the fungidb reduced set (a.a. length > 500) to the ref_protein dataset. To do so, I used the following commands
    ```bash
    $ mkdir blastdb; makeblastdb -in XP_028889033_homologs_refprot.fasta -parse_seqids -dbtype prot -title XP_028889033_refprot -out blastdb/XP_028889033_refprot
    $ blastp -db ./blastdb/XP_028889033_refprot -query XP_028889033_homologs_fungidb_use.fasta -outfmt "6 qseqid sseqid qlen slen pident mismatch score bitscore evalue" -max_target_seqs 1 -num_threads 4 -out XP_028889033_fungidb-refprot-blast.txt
    ```
    Explanation
    - -outfmt 6: tabular output, no comments
    - -max_target_seqs 1: only output one (best-scoring) match per sequence
      - **update 2021-06-17**: this option should be avoided as it doesn't do what it sounds like. See [this paper](https://academic.oup.com/bioinformatics/article/35/9/1613/5106166)
      - In the new analysis (script in the rmarkdown), I used `-evalue 1e-180` instead.
    - -num_threads 4: use 4 cpus to perform the search

## 2020-07-22 [HB] Identify homologs in Nakaseomyces
### Motivation
The original blast to both the refseq_protein and fungiDB databases yielded no hits in the well represented _S. cerevisiae_ _sensu stricto_ or _sensu lato_ clade. The only hits were in _C. glabrata_ and _N. castellii_. I'm particularly curious why the other Nakaseomyces group species, e.g. _C. bracarensis_, _N. dephensis_ and _C. nivariensis_ had not hits. Turns out even the NCBI nr_protein database doesn't contain any protein entries for the Nakaseomyces -- I verified this by blast'ing Pho4 protein sequence against the nr_protein and limited the organisms to Nakaseomyces. I then found / remembered that the [Genome Resource for Yeast Chromosomes](http://gryc.inra.fr/index.php) site contains the Nakaseomyces genomes. I verified this by repeating the Pho4p blast. I then blast'ed the first 500 a.a. of XP_028889033 in GRYC, selecting the Nakaseomyces (6 sps)as well as _S. cerevisiae_ (1), Lachancea (12), Naumovozyma (1, _N. castellii_), Yarrowia (3 _Y. lipolytica_ strains) 

### Approach
Got 15 hits from the GRYC blast. Downloaded the fasta sequence and the blast text output. The latter requires a lot of parsing, and there is no option that I can find to change the output format to a table. Instead, I just brutal-forced it -- copy and paste the table on the html page, put it into a text file, and edited it with vim (only 15 rows, not too bad). I also added an ID column to render the sequence ID more in-line with what I have for the other sequences.

I revamped the `blast.Rmd`. In the process of filtering and integrating the new hits, I found that I didn't properly filter the refseq_protein hits with the same length threshold I applied to the fungidb hits. So now I made the analysis consistent with respect to the selection criteria, and removed the _D. rugosa_ sequences (the reason is documented in the `README` files in the `output/gene-tree` folder or subfolders therein). In the end we get 100 sequences in total.
## 2020-08-06 [HB] Identify potential homologs in bacteria and in _S. cerevisiae_
HMMER and BLAST search for homologs of the N-terminal domain (350 a.a.) of XP_028889033 in viruses and bacteria

    MAFNFVRGWLLLAFYLSATWALTITENTVNVGALNIKIGSLTINPGVYYSIVNNALTTLGGSLDNQGEFYVTSANGLAASVSIVSGTIKNSGDLAFNSLRASVISNYNLNSIGGFTNTGNMWLGISGYSLVPPIILGSATNWDNSGRIYLSQNSGSASTITISQTLGSITNDGSMCIERLSWLQTTSIKGAGCINLMDDAHLQLQISPWSVSNDQTIYLSSSSSMLSVLGLSQSITGTKTYNVVGFGDGNSIRVNTGFSGYSYEGDTLTLSFFLGLFKIAFKIGTGYSKSGFSTNGLFGAGTRISYSGAYPGTVPDVCKCFDFPEPTTTPLPSSTSQSSKPSSSSSVIT

restricting the taxonomy to viruses, archaea and eubacteria, and e-value cutoff 0.01  No hits were found using either the PHMMER or JACKHMMER algorithm.

I repeated the search using blastp with e-value cutoff of 10, and taxonomy restricted to the same groups as above. The database in this case is the non-redundant proteins. This time I did get 3 significant hits!

![blastp nr hits](img/20200806-blastp-bacteria-nr-hits.png)

| query acc.ver | subject acc.ver | % identity | alignment length | mismatches | gap opens | q. start | q. end | s. start | s. end | evalue | bit score | % positives |
| --------------|-----------------| ---------- | ---------------- | ---------- | --------- | -------- | ------ | -------- | ------ | ------ | --------- | -----------|
| XP_028889033 | [PYD84265.1](https://www.ncbi.nlm.nih.gov/protein/PYD84265.1/) | 57.609 | 184 | 77 | 1 | 136 | 318 | 3 | 186 | 5.63e-70 | 226 | 76.63 |
| XP_028889033 | [WP_146232083.1](https://www.ncbi.nlm.nih.gov/protein/WP_146232083.1) | 88.372 | 86 | 10 | 0 | 6 | 91 | 6 | 91 | 2.88e-40 | 146 | 95.35 |
| XP_028889033 | [CQB89545.1](https://www.ncbi.nlm.nih.gov/protein/CQB89545.1) | 35.912 | 181 | 96 | 7 | 166 | 330 | 1 | 177 | 5.17e-16 | 89.4 | 51.93 |

As one can see from the query coverage and evalue columns, the first and second matches are quite significant. Both hits are from the species _Pseudomonas syringae_, which belongs to the class gammaproteobacteria. The third hit is from _Chlamydia trachomatis_, which belongs to a completely different phylum, Chlamydiae. Since it is a lot less similar to our query, we will focus just on the first two.

The second hit, while short, has high sequence identity. Wondering why I got 3 bacterial hits -- I expect either none or a lot -- I took the sequence of the first hit and repeated the blastp search. This produced two significant hits, including itself and the 3rd hit above. What does this mean? Are these highly species-specific sequences coming from fungi? Are they ancient proteins that have been lost in many many bacteria except for a few? Could these be annotation errors, namely the sample used to identify these bacterial sequences may be contaminated with fungal material?

## 2020-08-09 [HB] Homologs in Bacteria?

while reading about the [Pfam family (PF11765)](http://pfam.xfam.org/family/PF11765), I noticed that the [description](http://pfam.xfam.org/family/PF11765#tabview=tab0) said that this domain is **specific to fungi**. Does this mean the curator(s) of this family don't believe the bacterial sequences are true hits? From the PFam site I navigated to the linked [InterPro page](http://www.ebi.ac.uk/interpro/entry/InterPro/IPR021031/taxonomy/uniprot/#tree). There I learned that of the 561 members, 530 are in fungi -- in fact, 500/528 are in Saccharomycotina -- and only 31 are from bacteria. The hits in bacteria are almost all (30/31) in the alphaproteobacteria group, which is a different group compared to the gammaproteobacteria that _Pseudomonas syringae_ belongs to (see blast results above). 

![Sequence alignment of bacterial hits to XP_028889033 NTD](img/20200809-PF11765-phylogenetic-distribution.png)

The other suspicious sign is when I blast the _Pseudomonas syringae_ hit, which is 186 a.a. long and represents a "partial CDS", to all proteins labeled as _Pseudomonas syringae_ in the nr database, only the query itself came up in the hits. This is unexpected as the species is well studied as a plant pathogen and there must be a large number of well-assembled genomes in the species. At this point I have two theories explaining the blastp hits:

1. The hits represent false positives, likely due to fungal contamination of the bacterial sample.
1. The hits represent true sequeces in a _particular_ strain of the bacterium, possibly as a result of horizontal gene transfer from fungi.

![conclusion of the protein family being fungal specific](img/20200809-XP_028889033-nr-bacteria-hits-likely-spurious.png)

Despite all the suspicious signs and the fact that the two database searches above yielded hits from bacterial species that are very distant from each other, I don't think we can completely rule out the possibility of a more ancient origin of the domain (alternatively, this could result from convergent evolution). A paper studying the Hyr1 protein in _C. albicans_ showed that it is structrually similar to a bacterial adhesin. The bacteria that they identified are of the species _A. baumannii_, which belongs to gammaproteobacteria. But the fact that we get so few hits that are inconsistent between databases suggests that whichever hypothesis is true, the similarity is very low and the Hyphal_reg_CWP domain that we are concerned about is yeast specific.

## 2020-08-19 [HB] Homologs in _S. cerevisiae_?

I went back to the "species" tab on the [Pfam site for (PF11765)](http://pfam.xfam.org/family/PF11765#tabview=tab7) to check for any bacterial members. There indeed are, but the listed species are different from either the InterPro or my ncbi blast results, again raising questions about whether these bacterial hits are real or due to fungal contamination.

![pfam site taxonomy](img/20200819-pfam-taxonomy-view-PF11765.png)

Another notable finding is that the PFam species page listed two hits in _S. cerevisiae_: CSS1/YIL169C and HPF1/YOL155C. I used BLASTP-align-two-sequences feature to align them and also the two _C. glabrata_ homologs identified before to the NTD of XP_028889033 (as query) and found that they do appear to share some homology, although with very low sequence identity (~28%). See results below

![BLASTp graphic summary](img/20200821-Sc-Cg-blastp-XP_028889033-graphic-summary.png)
![BLASTp table](img/20200821-Sc-Cg-blastp-XP_028889033.png)

BLAST alignments for the _S. cerevisiae_ sequences can be viewed below:

[YIL169C](img/20200819-YIL169C-alignment.png)
[YOL155C](img/20200819-YOL155C-alignment.png)

While they do share sequence similarity with the NTD in XP_028889033, their domain architectures are quite different.

![domain architecture](img/20200819-PF11765-S-cerevisiae-domain-architecture.png)

Here is the blastp alignment for one of the two _C. glabrata_ homologs, XP_447567.2 ([CAGL0I07293g](http://www.candidagenome.org/cgi-bin/locus.pl?dbid=CAL0130316)), which was annotated as adhesin-like cell wall protein. Both the query coverage and identity are better than the _S. cerevisiae_ hits

[CAL0130316](img/20200819-CAGL0I07293g-alignment.png)

Another way to visualize the similarity/difference between the domain sequences from the _S. cerevisiae_ and _C. glabrata_ sequences and XP_028889033 is a dotplot. Below is a polydot plot produced with the [`flexidot_v1.06`](https://github.com/molbio-dresden/flexidot) script with word size of 7 and allowed substitutions of 2.

```bash
python flexidot_v1.06.py -i PF11765_domains_only.fasta -p 2 -t 0 -k 7 -S 3
```

![polydot plot](analysis/Sc-Cg-members/Polydotplot_wordsize7_S2.png)

To gain more insight into the potential functions of the two _S. cerevisiae_ hits, I looked at their annotations on SGD:

> YIL169C: protein of unknown function, secretd when constitutively expressed; ... S/T rich; putative glucan alpha-1,4-glucosidase.
> YOL155C: Haze-protective mannoprotein; reduces the particle size of aggregated proteins in white wines

The latter has some obvious connection to adhesion, based on its role in aggregation. The former hits two features seen also in the adhesins: S/T rich region and putative gluan alpha-1,4-glucosidase activity, but seems to be not anchored on the cell wall (or secreted). Prediction of GPI-anchor using two programs (see `Sc-Cg-members/README.md`) suggests that both may be cell-wall anchored. Given the location of the domain in the _S. cerevisiae_ genes (in the middle) and their likely cell wall localization, I suspect their NTDs are not exposed but rather buried in the cell wall matrix and may be involved in modifying the cell wall composition (with the glucan alpha-1,4-glucosidase activity).

Based on the above results, I'm further convinced that the PF11765 domain is indeed an ancient one. This protein family has either been lost or evolved to perform different functions in most of the Saccharomycetaceae species, but has dramatically expanded in the MDR clade and _C. albicans_ clade to function as adhesins.

I further asked if using the _C. glabrata_ domain sequence as query, I can recover hits in _S. cerevisiae_ and other Saccharomycetales yeasts. **The answer is no.**
## 2020-09-13 [HB] Homologs in other _C. auris_ proteomes
This analysis stems from Jan's question of what other homologs are there in the five _C. auris_ proteomes. To answer this question, I did the following:

1. Bring over the proteome fasta files from the global analysis folder
    ```bash
    mkdir data/Cauris-strains; cd data/Cauris-strains
    ln -s ../../../../01-global-adhesin-prediction/data/proteome-fasta/*Cand_auris* ./
    cd ..
    ```

1. Construct blast database
    ```bash
    cat Cauris-strains/*.gz > Cauris-strains/Cand_auris_five_strains_protein.faa.gz
    gunzip -c Cauris-strains/Cand_auris_five_strains_protein.faa.gz | \
        makeblastdb -in - -parse_seqids -dbtype prot -title Cand_auris_five_strains -out blastdb/Cand_auris_five_strains
    blastp -db ./blastdb/Cand_auris_five_strains -query XP_028889033_query.fasta -max_hsps 1 -outfmt "7" -num_threads 4 -out ../output/XP_028889033-Cauris-five-strains-blast.txt
    ```

1. Edited the output text file by adding a header. Then open that file in Excel and removed the comment lines along with several entries with E-value > 10E-5. The result is stored in an excel file of the same base name as above.

1. Extract all blast-identified _C. auris_ homologs sequences using my custom script:
    ```bash
    cd Cauris-strains
    awk '$1 ~ /^XP_028889033.1_PF11765/ && $4 > 60' XP_028889033-Cauris-five-strains-blast.txt | \
    | cut -f2 | sort > list-Cauris-five-strains-homologs-blast.txt
    python ../../script/extract_fasta_gz.py Cand_auris_five_strains_protein.faa.gz list-Cauris-five-strains-homologs-blast.txt cauris-five-strains-homologs.fasta
    cd ../../07-Cauris-polymorphism/input; ln -s ../../02-blast/data/Cauris-strains/cauris-five-strains-homologs.fasta ./
    ```
## 2020-10-20 [HB] Correct GRYC mistakes
During a discussion Rachel pointed out that the domain architecture figure showed a few sequences that are shorter than 500 a.a. I doubled checked and found that there are two of them. One is from _N. delphensis_ and in my notes I explained the reason why I included it, because it is a "partial CDS". The other is from _N. bracarensis_. It turned out that the length of the protein in the blast hit table is incorrect. I manually edited that file and most likely introduced the error in the process. So I just removed the latter in the new `XP_028889033_homologs_combine.fasta`. Also, I noticed that I included one sequence from _C. auris_ strain B8441, making _C. auris_ the only species with more than one strain represented in the homologs list. Moreover, I didn't systematically include _all_ hits from B8441. So I decided to remove that. Lastly, I decided to include CAGL0L00227g, which was originally excluded because query coverage of this hit was 47%, below the 50% cutoff I set. However, upon further looking, I found this sequence interesting as it is very long (~3kb, similar to the query) and has extremely high Serine content. Thus I decided to include it to demonstrate the evolution of this protein family in _C. glabrata_.

**Update 2020-11-05**
I reversed my decision and removed CAGL0L00227g. The reason is because when I included it in the homologs list, the alignment and gene tree analysis suggested that it is far removed from the existing homologs in the Nakaseomyces, and will throw off the gene tree. This suggested to me that this gene likely evolved from a more ancient duplication. If I want to include it in the tree and properly interpret the topology, I would need to lower my query coverary and/or e-value cutoff to include more potential homologs.
## 2021-01-09 [HB] Additional homologs through blastp against refseq_protein
While writing up the blast results, I repeated the blast search on ncbi against the refseq_protein, and found that more species are found in the hit list, likely because the database has grown over the past few months. But the results won't change the major conclusions. Below is the taxonomy list of all species that contain hits with the same criteria (e-value < 1e-5 and query coverage > 50%)
![](img/20210109-blastp-N560-refseq-all-species-phylogeny.png)

The red arrows point to species excluded from my homologs list. Among the ones I excluded, the following three are notable for different reasons:

1. the single fission yeast hit ([XP_013021368](https://www.ncbi.nlm.nih.gov/protein/891576397)) from _Schizosaccharomyces cryophilus_. It has decent query coverage (76%) and somewhat low but not terrible percent identity (~28%). The protein description page did describe it as a hyphally regulated cell wall protein, based on the presence of the PF11765 domain. If verified, this would suggest the protein family originated at the root of all Ascomycetes.
1. _Kazachstania africana_ is within the Saccharomycetacea and had 8 hits, more than any other species in the genus, including _C. glabrata_. If verified, this would suggest an independent expansion in the Saccharomycetaceae, in addition to the two expansions in the MDR and _albicans_ clade.
1. _Candida orthopsilosis_ is most closely related to _C. parasiolosis_ and next closest to _L. elongisporus_, both of which harbored significantly fewer homologs than the neighboring _albicans_ clade (~3 vs > 10).

In conclusion, the omission of the above species do not alter the main conclusions reached so far, except for the possibility that the PF11765 domain originated earlier at the root of the fission and budding yeasts.

**Update 2021-01-21** Hits in fission yeast and bacteria are questionable

The fission yeast hit comes from a protein that is only 393 aa long, while the bacterial one is 91 amino acid. I took those two proteins and blast'ed them against the nr_protein database, restricting the taxonomy to Schizosaccharamycetes and eubacteria respectively, maintaining the same 1e-5 e-value cutoff. In both cases, the only hit recovered is the query protein itself. This suggests to me that the original hits are not worth following up with as they represent lone hits that are likely due to sequencing or annotation errors.

## 2021-01-23 [HB] Repeat blast searches with N-360 aa from XP_028889033 (for writing up the results)

_Issue_

While writing up the results, I realized that my original search was done with the first 560 amino acids from XP_028889033, which includes both the N-terminal PF11765 domain (12-327) and also one the Hyr1 domain (PF15789) repeats. This alone wouldn't change the search result by a lot. But when I later applied the 50% query coverage filter, I was effectively applying a more stringent cutoff, because most of the hits only share similarity with the PF11765 domain and not the Hyr1 domain.

_Investigation_

I repeated the search using the first 360 amino acids from XP_028889033. The resulting hit table and fasta are downloaded. To distinguish these files from the original ones, I added the "N560" and "N360" suffixes. The original search with N-560 aa yielded 190 sequences, of which 154 passed the 50% query coverage cutoff. I excluded 9 species from this list (see notes above) and arrived at the final list. Below is the taxonomy for all hits (not filtered by query coverage cutoff). Species labeled by a red arrow are those not included in the final analysis.

![N360 taxonomy](img/20210124-blastp-N360-refseq-all-species-phylogeny.png)

I then compared the new list with the old one (see `blast.Rmd`) and decided to add 7 sequences to the original list. See `blast.Rmd` for details.

_Procedure_

I repeated all three searches (refseq_prot, GRYC, fungidb) with the N360 amino acids as query, so as to make the writing a little easier. The results didn't change for GRYC and fungidb, but a few more sequences are identified in the blastp search against the refseq_prot, which I will incorporate into our analysis.

## 2021-01-31 [HB] Vary e-value cutoff with N-360 aa from XP_028889033

_Notes_

While re-assembling the list of homolog sequences using the new N360 search results, I found that one of the original sequences, XP_001383953.2 from _S. stipitis_, is no longer in the new list. I then repeated the blastp search with the N360 query, relaxing the e-value cutoff from 1e-5 to 1e-3, and lo-and-behold, it's at the bottom of the list. I decide to leave it out so all the results are now based on the N360 with e-value < 1e-5, query coverage > 50% and length > 500 aa. It is of interest to note that changing the e-value from 1e-5 to 1e-3 only added four sequences (see below).

![new-seqs-loose-e-value](img/20210131-blastp-e-value-1e-3-seqs.png)

See `blast.Rmd`'s relevant section for more discussion.


## 2021-03-01 [HB] _C. auris_ clade III should have 8 members in this family
While reading Muñoz 2020 Genetics, I noticed that they identified a total of 8 members of this family in Clade I and III of _C. auris_. Since the type strain we used for our analysis comes from Clade III (B11221), I wonder why we missed one.

![iff member in cauris](./img/20210301-munoz-2021-genetics-iff-family-in-cauris.png)

To investigate this, I performed blastp using the N-terminal 330 amino acids (using 360 should give the same result) against the non_redundant protein database on NCBI, restricting the taxonomy to _Candida auris_ (taxid:498019), and setting e-value cutoff to 0.05 (although everything that was returned had extrememly significant e-values).

![blastp](./img/20210301-blastp-nr-identify-B8441-8th-member.png)

As can be seen in the table above (only B8441 - B9J08 and B11221 - CJI97 hits were shown), one B8441 hit doesn't have a corresponding B11221 hit. The ID of it is B9J08_004098. The reason CJI97 had two hits per B9J08 hit is because the nr database contains both the refseq and non-refseq proteins. B11221 is the only _C. auris_ strain with a Refseq assembly, hence the duplicates.

Looks like I might have to do the blastp again, this time adding the B9J08_004098 back to the homologs table. This will minimize the needed changes to other parts of the results, which were all based on our query protein XP_028889033 from B11221.

## 2021-04-03 [HB] Missing homolog in B11221
I subsequently did more blastp search to figure out why I'm missing the 8th member of the family in the clade IV strain B11221. I found several relevant things:

1. If I use B9J08_004098's PF11765 domain (11-326) as query and blast the B11221 translated CDS (or protein) database, I could recover the seven identified previously and also three more hits. Two of them have very short alignment length (less than 50 amino acids), but a third one has 146 amino acids aligned, with ~20.5% identity. I then looked up this sequence in the ref_protein database by its ID (XP_028890323) and found it to be only 177 amino acids long. I did follow the link and found the corresponding gene (NW_021640165) and it appears to be a complete CDS with a stop codon. So it's still unclear why I only found 7 members in the B11221 genome, even though the latest Muñoz et al 2021 Genetics paper showed 8 (Figure 6).

_Discussion (2021-04-19 [HB])_

Continuing to investigate the reason for the missing gene in B11221: this time I blast'ed the entire protein sequence using `tblastn` against either the B8441 or the B11221 assembly, using the default settings (e-value 0.05, word size 6, blosum 62 matrix, gap 11/1). This led to the following discoveries:

- B9J08_004098 is located on scaffold01 in B8441 genome. According to the [supplementary table 1](https://gsajournals.figshare.com/articles/dataset/Supplemental_Material_for_Mu_oz_et_al_2021/13759276?file=26384548) of the Muñoz _et al._ 2021 (PMID: 33769478), scaffold00001 along with 00008 map to chromosome 1.

    ![B8441](./img/20210419-B9J08_004098-blast-in-B8441.png)

- `tblastn` against the B11221 genome resulted in the following hits (graphic summary)
  
    ![B11221 graphic](./img/20210419-B9J08_004098-blast-in-B11221-graphic.png)

    The first three lines are continguous sequences matching the N-terminus and to a less extent the C-terminus, and they are almost surely among the other 7 homologs. The last line is interesting in that it is split into two discontinuous sequences with very high similarity. I checked the scaffolds they came from and they are from scaffolds00001 and 00015, where the former maps to chromosome 1 while the latter is not assembled into the genome. Notably, the match to scaffold00001, which corresponds to the C-terminus of the protein, is nearly 100% identical, with one amino acid mismatch that could result from technical artifacts. I thus determine that the missing Hil4 homolog in B11221 is located on chromosome 1 (scaffold0001) and is located at the 3' end of the sequence (the coding sequence is on the minus strand, explaining why the N-terminal region is missing). Note that Hil4 in B8441, which is PIS52481.1, sits on chromosome 1. So this further convinces me that Hil4 is indeed present in B11221.

    ![scaffold00001 alignment](img/20210621-B9J08_004098-blast-in-B11221-scaffold00001-align.png)

    ![scaffold00015 alignment](img/20210621-B9J08_004098-blast-in-B11221-scaffold00015-align.png)
    
    **Update 2021-07-09 [HB]** while analyzing PIS58185 in `01-global-adhesin-prediction`, I landed on a partial CDS entry XP_028891378.1. It is on chromosome 1 and matches PSK79349 (=Hil4 in B11243)  from residue 775 to residue 1247 almost perfectly (468/473 residues are identical). I thus believe that XP_028891378.1 is part of the Hil4 ortholog in B11221. But because it is missing the N-terminal PF11765 domain (which is on the unassembled scaffold00015), previous searches using the NTD failed to identify it.
    
    My analysis above led me to the following conclusions:
    
    1. Hil4 homolog is present in B11221 and is located at the 3' tip of chromosome 1 (scaffold00001).
    2. Chromosome 1 in B11221 is incomplete at the 3' tip and there doesn't appear to be an unassembled scaffold that corresponds to that piece. Instead, it may just be missing.
    3. For my downstream chromosomal location analysis, it would be convenient to convert the B9J08_004098 locations to the locations in the B11221 genome. What I can do is to modify the chromosomal information during that analysis to correspond to the scaffold0001 hit here.

## 2021-07-05 [HB] _C. auris_ B11245 homologs
As I work through Rachel's analysis to identify additional proteins in _C. auris_ that share the features of the Hil family, I found that she swapped B11245 for B11243 as the representative for Clade IV. This is partly because B11245 was sequenced and assembled to a more complete degree. According to Rachel's notes, most of the sequences in the two strains are identical. However, when I try to determine if a sequence in Rachel's analysis is part of the Hil family, I realized that I don't have the IDs for the B11245 homologs. Hence I need to perform blast on this strain separately, following the procedures below:

```bash
# prepare blast database
gunzip -c Cauris-strains/Cand_auris_five_strains_protein.faa.gz | makeblastdb -in - -parse_seqids -dbtype prot -title B11245 -out blastdb/Cand_auris_B11245
# use XP_028889033's PF11765 domain to identify all Hil homologs in B11245
blastp -db ./blastdb/Cand_auris_B11245 -query Cauris-strains/B11243-homologs.fasta -max_hsps 1 -evalue 1e-180 -outfmt "7" -num_threads 8 -out ../output/B11245-Hil-homologs-blast.txt
# to identify for each Hil homolog in B11243 its ortholog in B11245
# we extract the 8 Hil homologs from B11245 as identified above and use the 
# blast2seq function to correspond them with those in B11243
bioawk -c fastx '$name ~ /PSK/{print ">"$name;print $seq}' Cauris-strains/cauris-five-strains-homologs.fasta > Cauris-strains/B11243-homologs.fasta
python ../../script/extract_fasta_gz.py GCA_008275145.1_Cand_auris_B11245_protein.faa.gz B11245-homologs-id.txt B11245-homologs.fasta
# submit the two sequence fasta files to the blast 2 seq app on ncbi

# also blast the 8 Hil proteins from B11243 in the proteome database of B11245
blastp -db blastdb/Cand_auris_B11245 -query Cauris-strains/B11243-homologs.fasta -evalue 1e-180 -use_sw_tback -soft_masking true -outfmt "7" -num_threads 8 -out ../output/B11245-Hil-homologs-blast.txt
# Hil 8 homolog appears to be missing...
```

From the result we can assemble the following:

| Hil | B11243 | B11245 |
|-----|--------|--------|
| 1 | PSK76857 | QEL63125 |
| 2 | PSK74847 | QEL62736 |
| 3 | PSK76051 | QEL59874 |
| 4 | PSK79346 | QEL57973 |
| 5 | PSK75908 | QEL61272 |
| 6 | PSK79348 | QEL57971 |
| 7 | PSK79720 | QEL62783 |
| 8 | PSK76858 | missing? |

From the B11221 Hil8 ortholog (XP_028889034), I found that this gene is expected to be located on chromosome 6. When I used PSK76858 as the query and performed tblastn against the B11245 genome, I can identify nearly the entire gene in broken pieces on its chromosome 6, suggesting that the reason it is missing in the blastp search is because the incomplete assembly of that chromosome, which led to the protein to be unidentified.

![B11245 Hil8](img/20210706-B11245-missing-Hil8-on-chromosome6-tblastn.png)

## 2021-10-17 [HB] PSI-BLAST

While attending Jan's lecture on BLAST, I realized that in my early try of the PSI-BLAST algorithm, I didn't use it properly -- for PSI-BLAST to realize its power, one needs to run the search iteratively. The idea of PSI-BLAST is to use the hits from the previous round (first round is just a regular blastp search) to build a PSSM and then search with that PSSM to achieve higher sensitivity.

I thus redid this analysis using the N360 amino acids from Hil1 in _C. auris_, setting the E-value cutoff at 1e-5. I then required >30% identity for a hit to be included in building the PSSM for the second round. Using this approach, I could identify the Hyphal_reg_CWP domains in the Saccharomycetaceae species, such as _S. cerevisiae_. They are the ones that I commented on before -- the amino acid sequences are < 30% identical and the protein domain architectures are very different, with the Hyphal_reg_CWP domain in the middle instead of in the N-terminal of the proteins.

## 2021-10-24 [HB] Als blast
I'd like to include the homologs count in Figure 1 for the Als family as well. To do so, I followed a similar recipe as I did for the Hil family: I chose to use _C. albicans_ Als3's NTD as the query. Specifically, I used the refseq protein sequence (XP_710435.2) from aa 53-298 (based on the [Pfam table](http://pfam.xfam.org/protein/Q59L12#tabview=tab0) identifying its Candida_ALS_N domain). I used blastp to search this query against the refseq database with E-value cutoff of 1e-5. Query coverages are all above 50%. I also referenced three other publications that contained partial list of homologs for the Als family, namely Butler _et al._ 2009 (PMID: 19465905), Muñoz _et al._ 2018 (PMID: 30559369) and Linder and Gustafsson 2008 (PMID: 17870620). Among the three, the Muñoz 2018 paper appears to have a more stringent criteria for identifying homologs, as its numbers are often much smaller than the other two and my blastp result. The other two papers broadly agree with my blastp results. I also did the same blast on GRYC for the Nakaseomyces group. The outputs from both searches are stored in the `data` folder under their respective subfolder, along with the output for the Hil family. The consolidated table is stored in the `output` folder and soft linked to the `04-homologs-property` folder.

## 2022-04-06 [HB] Address RC reviewer 1 comments

The main questions/concerns raised are:

1. Hil1 PF11765 domain sequence (XP_028889033) was used as bait. Will the result change if a different homolog was used?
   - Try Hyr1 and *C. glabrata* homolog as bait
2. The criteria for excluding species from the BLAST result is arbitrary
   - Include all species that exist in the Shen et al 2018 Y1000 tree
   - Use the result to repeat the gene family expansion analysis
3. Useful to know the number of homologs beore and after applying the requirement for conserved position of the PF11765 domain in the protein
   - Record the number of homologs before and after the filtering

BLAST strategy:

- blastp with E-value cutoff of 1e-5
- low complexity filter on
- minimum query coverage 50%

Did:

- Extracted amino acid sequences corresponding to the annotated PF11765 domain in Caur Hil1 (XP_028889033), Call Hyr1 (XP_722183.2) and Cgla Hyr1 (XP_445977.1, CAGL0E06600g).
- Performed web-based blastp search with the above parameters against the refseq_prot database.
- Downloaded the results in the BLAST archive format 
  `blast_formatter -rid 4VFCXY5F016 -out 20220406-expanded-PF11765-blastp-out.asn -outfmt 11`
- Downloaded and extracted the taxonomy database
  `wget ftp://ftp.ncbi.nlm.nih.gov/blast/db/taxdb.tar.gz; tar xzvf taxdb.tar.gz` in the same folder as above
- Reformatted BLAST output to extract subject ID, length, query coverage, % identity, subject scientific name
  `blast_formatter -archive 20220406-expanded-PF11765-blastp-out.asn -out expanded-PF11765-blastp-out.txt -outfmt "7 qseqid qcovs sseqid qstart qend slen sstart send pident evalue staxids sscinames"`
- To expand the search beyond the refseq_prot database, I repeated the search against the new "clustered nr" database
  `blast_formatter -rid 4VFCXY5F016 -out 20220406-expanded-PF11765-blastp-out.asn -outfmt 11`

Results

- See [here](https://rpubs.com/emptyhb/c037-expanded-blast-analyses) (html output of Rmarkdown file [here](analysis/expanded-blast/expanded-blast.Rmd)).
- In short, the combined queries yielded just 3 new hits.

## 2022-04-20 [HB] Reviewer 3 comment: refseq_prot records complete?

### _C. tropicalis_

One of reviewer 3's criticism is that in their unpublished work, they found that seven of the _C. tropicalis_ Hil homologs listed in our manuscript are "incomplete", in the sense that the refseq annotated protein is not the full length product. When I looked closely at the refseq records, it turned out that the mRNA record indicated the product is indeed incomplete. I initially understood this as meaning the ORFfinder algorithm was unable to identify the stop codon. But later I realized that this is not necessarily the case. Note that the "completeness" field in the mRNA record can take several values, including "incomplete 5' end", "incomplete 3' end" and "incomplete on both ends" (exact wording may differ). What "incomplete 5' end" meant was that the gene finder wasn't able to identify an **upstream stop codon** (for the upstream gene) 5' of the current gene's start codon. Similarly, "incomplete 3' end" means that the pipeline cannot identify the **start codon** of the downstream gene. Missing stop codon should, in theory, result in failed gene annotation. But I decided to check the _C. tropicalis_ hits more closely anyway. Specifically, I blasted the mRNA sequence against a [new genome assembly](https://www.ncbi.nlm.nih.gov/assembly/GCA_013177555.1) (for the same MYA-3404 type strain) based on a mixture of PacBio and Illumina reads. My reasoning is that if the CDS is at least complete, I should be able to locate both the start and the stop codon in the genome sequences (especially the stop codon).

| protein_ID     | length | mRNA_ID      | compared_to_new_assembly                                     |
| -------------- | ------ | ------------ | ------------------------------------------------------------ |
| XP_002547764.1 | 1405   | XM_002547718 | 10 SNPs, stop codon present                                  |
| XP_002550900.1 | 665    | XM_002550854 | 1 SNP and 1bp deletion near annotated stop codon; ORF finder identifies a **1566 aa product** |
| XP_002549927.1 | 925    | XM_002549881 | 2 SNP and 1bp deletion near annotated stop codon; ORF finder identifies a **1244 aa product** |
| XP_002546360.1 | 1952   | XM_002546314 | 1 SNP and 1bp deletion in the middle of the CDS; ORF finder identifies a **1428 aa product** |
| XP_002545452.1 | 1383   | XM_002545406 | 1 SNP, stop codon present                                    |
| XP_002547620.1 | 1689   | XM_002547574 | 10 SNPs, stop codon present                                  |
| XP_002547617.1 | 827    | XM_002547571 | **Protein is labeled as "no-right"**; poor match towards the C-terminus, 1bp deletion at least; ORF finder identifies a **2346 aa product** |
| XP_002547615.1 | 749    | XM_002547569 | Multiple small indels and SNPs towards the C-terminus; ORF finder identifies a **370 aa product** |
| XP_002545510.1 | 656    | XM_002545464 | Many SNPs at the C-terminus, 3 single bp indel, stop codon not present, ORF finder identifies a **1990 aa product** |
| XP_002547616.1 | 1626   | XM_002547570 | Poor match in the middle and towards the C-terminus, multiple indels, stop codon not present, ORF finder identifies a **247 aa product** |
| XP_002547904.1 | 1038   | XM_002547858 | Perfect match                                                |
| XP_002550128.1 | 1230   | XM_002550082 | Perfect match                                                |
| XP_002550768.1 | 455    | XM_002550722 | two SNPs, stop codon not present, ORF finder identifies a **1455 aa product** |
| XP_002546744.1 | 328    | XM_002546698 | Perfect match                                                |
| XP_002547891.1 | 669    | XM_002547845 | Perfect match                                                |
| XP_002550387.1 | 588    | XM_002550341 | two SNPs, stop codon present                                 |

To see whether this is a general pattern for the other genomes, I also looked at several other species.

### _C. albicans_

For _C. albicans_ SC5314, the reference assembly was updated in 2016 (the strain was resequenced in 2013, PMID: 24025428). Based on the publication, the assembly should be of high completeness and quality. Notably, none of the _albicans_ hits were annotated as incomplete for the protein product.

Since no newer assembly using 3rd gen technology exists for the same strain, I blasted the 12 mRNA records against two newer assemblies, [GCA_017309835.1](https://www.ncbi.nlm.nih.gov/assembly/GCA_017309835.1) and [GCA_005890745.1](https://www.ncbi.nlm.nih.gov/assembly/GCA_005890745.1). The former is for the CHN1 strain, sequenced using a mixture of Oxford Nanopore and MiSeq, assembled using Flye v.2.7.1 and Pilon v1.23. The latter is for the NCYC4166 strain, sequenced PacBio Sequel and assembled using FALCON-Unzip v. 1.1.3.

| protein_ID  | length | mRNA_ID   | CHN1                            | NCYC4166                       |
| ----------- | ------ | --------- | ------------------------------- | ------------------------------ |
| XP_722183.2 | 919    | XM_717090 | 96% identical, 26 gaps, 921 aa  | 97% identical, 38 gaps, 927 aa |
| XP_714203.1 | 1249   | XM_709110 | >1 HSPs, 1222 aa full length    | >1 HSPs, 1077 aa full length   |
| XP_717330.2 | 941    | XM_712237 | 98% identical, 6 gaps, 943 aa   | 99% identical, 3 gaps, 942 aa  |
| XP_715137.2 | 941    | XM_710044 | 97% identical, 6 gaps, 943 aa   | 99% identical, 3 gaps, 942 aa  |
| XP_711686.2 | 1562   | XM_706594 | >1 HSPs, 1222 aa full length    | >1 HSPs,  2186 aa full length  |
| XP_718630.1 | 1526   | XM_713537 | 99% identical, 1 gap (?) doubt  | 99% identical, 0 gap, 1526 aa  |
| XP_714201.2 | 714    | XM_709108 | 98% identical, 13 gaps, 539 aa  | 96% identical, 57 gaps, 695 aa |
| XP_715535.1 | 1308   | XM_710442 | 98% identical, 11 gaps, 1101 aa | 99% identical, 0 gap, 1308 aa  |
| XP_717969.1 | 1225   | XM_712876 | 99% identical, 0 gap, 1225 aa   | 99% identical, 0 gap, 1225 aa  |
| XP_714246.2 | 1085   | XM_709153 | 96% identical, 60 gaps, 893 aa  | 98% identical, 3 gap, 1086 aa  |
| XP_717774.1 | 511    | XM_712681 | 99% identical, 0 gap, 511 aa    | 99% identical, 0 gap, 511 aa   |
| XP_717775.2 | 1244   | XM_712682 | Partial match, 1155 aa          | >1 HSPs,  1162 aa full length  |

- The result shows that for _C. albicans_, the reference assembly hits are all fine.

### _C. parapsilosis_

For _C. parapsilosis_, which has the most hits shorter than the 500 aa cutoff (10/15), the reference assembly was submitted in 2011 and last updated in 2020, although it is still at contig level. The particular strain in that assembly, CDC317, has not been subject to another sequencing effort. I found a scaffold-level assembly for a different strain, CBS6318 ([GCA_000982555.2](https://www.ncbi.nlm.nih.gov/assembly/GCA_000982555.2)), which was completed in 2014 and last updated in 2019. I blasted all 15 hits against that assembly. For one of them (XM_036807488), I used ORFfinder to identify a 414 aa product with the same start codon position in the second strain's assembly, compared with 411 aa in the reference assembly. For another one (XM_036807487), the predicted size in the second assembly is 420 aa vs 414 aa in the reference one. **The rest all matched between the two assemblies**. So overall it seems like the short products are validated at least based two assemblies, both of which used short reads.

| refseq_pID     | refseq_tID   | refseq_len | new_qcov | new_%id | compared_to_new_assembly |
| -------------- | ------------ | ---------- | -------- | ------- | ------------------------ |
| XP_036665264.1 | XM_036808537 | 1671       | 100%     | 95.11%  |                          |
| XP_036664320.1 | XM_036807488 | 411        |          |         |                          |
| XP_036665262.1 | XM_036808535 | 1640       |          |         |                          |
| XP_036667771.1 | XM_036811321 | 420        |          |         |                          |
| XP_036664319.1 | XM_036807487 | 414        |          |         |                          |
| XP_036664321.1 | XM_036807489 | 419        |          |         |                          |
| XP_036668412.1 | XM_036812035 | 418        |          |         |                          |
| XP_036664318.1 | XM_036807486 | 439        |          |         |                          |
| XP_036667770.1 | XM_036811320 | 410        |          |         |                          |
| XP_036664322.1 | XM_036807491 | 437        |          |         |                          |
| XP_036663837.1 | XM_036806952 | 438        |          |         |                          |
| XP_036665265.1 | XM_036808538 | 1714       |          |         |                          |
| XP_036663836.1 | XM_036806951 | 404        |          |         |                          |
| XP_036662815.1 | XM_036810545 | 1429       |          |         |                          |
| XP_036665263.1 | XM_036808536 | 2273       |          |         |                          |

### _C. glabrata_

For _C. glabrata_, the Cormack lab has generated PacBio SRII based assemblies for several strains, including the genome reference strain CBS138. I used blastp to search the three hits based on the RefSeq assembly (GCF_000002545.3) against the new one (GCA_010111755.1) using a locally installed database and got the following result. 

| protein_ID     | length | mRNA_ID      | compared_to_new_assembly                             |
| -------------- | ------ | ------------ | ---------------------------------------------------- |
| XP_002999585.1 | 3241   | XM_002999539 | 100% identical to QHS68452.1                         |
| XP_445977.1    | 965    | XM_445977    | 100% identical to QHS65613.1                         |
| XP_447567.2    | 1771   | XM_447567    | 88.7% identical to QHS67215.1, which is 1861 aa long |

### Conclusion

Based on the above I can conclude that:

- _C. tropicalis_ is likely a specific case -- either its genome is particularly tough to sequence/assemble, or the original assembly was flawed. Other genomes I looked at don't have the same problems.
- The issues affecting _C. tropicalis_ can be summarized as follows: the PF11765 domain hit is credible (that's what we used as query in blast). However, the identified protein may be misannotated, meaning that it may be shorter or longer than it really is (in _C. tropicalis_'s case more likely to be shorter than longer, for reasons unclear to me).
- Because of the above, the issue *does not* affect the accuracy of the blastp hits. What it does affect is when I applied the 500 aa length filter, some proteins may have been wrongly left out. I plan to carefully review all hits that would be left out due to length in the revision and so this should be addressed.
- Note that none of our conclusions specifically hinged upon the Hil homologs in non _C. auris_ species being complete. The primary use of those sequences are 1) infer the evolutionary history of the family to demonstrate independent expansion, which relies only on the PF11765 domain alignment; 2) demonstrate the rapid divergence in the central domain (reviewer suggested changing this term, call it B-region?). For the latter, the incomplete proteins may affect the S/T frequency and β-aggregation potential analyses, but they are not expected to cause significant changes to any of the conclusions (which is that both features are highly variable among homologs).

### _C. auris_ strains

- Of particular interest is the quality of the hits in _C. auris_, including both the reference assembly and the other strains we used for various analyses. First of all, I checked the sequencing and assembly details for the genomes:

| Strain | Clade | Assembly_ID     | Sequencing_Technology | Assembly_Notes           |
| ------ | ----- | --------------- | --------------------- | ------------------------ |
| B11221 | III   | GCF_002775015.1 | PacBio                | HGAP v.3, scaffold       |
| B8441  | I     | GCA_002759435.2 | PacBio                | HGAP v.3, scaffold       |
| B11220 | II    | GCA_003013715.2 | Oxford Nanopore       | Flye v.2.4.2, complete   |
| B11243 | IV    | GCA_003014415.1 | Illumina              | Spades v.3.1.1, scaffold |
| B11205 | I     | GCA_016772135.1 | PacBio                | Canu v.1.6, complete     |
| B13916 | I     | GCA_016772235.1 | PacBio                | Canu v.1.6, complete     |
| B17721 | III   | GCA_016772175.1 | PacBio                | Canu v.1.6, complete     |
| B12037 | III   | GCA_016772215.1 | PacBio                | Canu v.1.6, complete     |
| B12631 | III   | GCA_016772195.1 | PacBio                | Canu v.1.6, complete     |
| B12342 | IV    | GCA_016772155.1 | PacBio                | Canu v.1.6, complete     |
| B11245 | IV    | GCA_008275145.1 | Oxford Nanopore       | Canu v.1.5, complete     |

- we will add these information to Table S6 and use them to support the validity of our within species analysis for the Hil family in _C. auris_.

## 2022-04-22 [HB] _Scheffersomyces Stipitis_ genome, new assembly

As I examined the completeness of the protein hits, I found that _S. Stipitis_ had a large number of incomplete protein products (14/16). I identified a more recent assembly sequenced using Oxford Nanopore ([GCA_016859295.1](https://www.ncbi.nlm.nih.gov/assembly/GCA_016859295.1/)). Fortunately the researchers generating this assembly also annotated the genome. I thus downloaded their protein fasta file, made a blast database for it

```bash
# in 02-blast/data/s_stipitis_assembly
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/016/859/295/GCA_016859295.1_ASM1685929v1/GCA_016859295.1_ASM1685929v1_protein.faa.gz
# make the blastdb
cat GCA_016859295.1_ASM1685929v1_protein.faa.gz | gunzip -c | makeblastdb -in - -parse_seqids -dbtype prot -title S_stipitis_NRRL_Y7124 -out ../blastdb/S_stipitis_NRRL_Y7124
# perform blastp
# I actually wrote a script in the assembly folder to do all the above and the blast. check that instead
```

I identified a total of 13 hits, of which seven are above the 50% query coverage threshold. This is quite different from the 18 hits identified in the refseq assembly. When I blasted the refseq hits against the new assembly, some of them had poor matches. Without further investigation, it is not clear what is the cause of the discrepancy. I plan to use the new assembly hits for downstream analyses given that it is more recently sequenced with a higher level of completeness.

## 2022-05-01 [HB] *C. glabrata* new assembly has more hits

_Reference_

Xu, Zhuwei, Brian Green, Nicole Benoit, Michael Schatz, Sarah Wheelan, and Brendan Cormack.  De Novo Genome Assembly of Candida Glabrata Reveals Cell Wall Protein Complement and Structure of Dispersed Tandem Repeat Arrays.  *Molecular Microbiology*, February 18, 2020. https://doi.org/10.1111/mmi.14488.

_Notes_

- The refseq assembly for the species was generated back in 2004 (Dujon 2004) using a combination of shotgun sequencing and Sanger sequencing. The latter was used to sequence Bacterial Artificial Chromosomes in order to complete the assembly. As such its quality is actually not bad.

- In their new effort, they used PacBio to provide the skeleton and used Illumina reads to polish the sequences. In order to **validate** the subtelomeric region assembly, they cloned the chromosome ends into fosmids and performed Sanger sequencing on those, and confirmed that their original assembly was correct.

- I downloaded the annotated protein fasta from the [assembly](https://www.ncbi.nlm.nih.gov/assembly/GCA_010111755.1/) and performed `blastp`. 13 hits were identified using either the Calb_Hyr1 or the Cgla_Hyr1 query. I took the protein IDs and used the batch entrez tool to look them up. I found out their systematic locus IDs (in the form of CAGL0xxxxx) and matched them against Table S5 in Xu, Cormack, et al 2020 (PMID: 32068314). I found they matched exactly the 13 annotated genes belonging to cluster V adhesins. Some of these were in the refseq assembly and are included in the 104 homologs list.
  | pID      | gID          | length | RefSeq | in104 | subtelomeric |
  | -------- | ------------ | ------ | ------ | ----- | ------------ |
  | QHS64595 | CAGL0B00154g | 956    | No     | No    | Yes          |
  | QHS64811 | CAGL0B05061g | 2232   | No     | No    | Yes          |
  | QHS65049 | CAGL0D00143g | 3331   | No     | No    | Yes          |
  | QHS65613 | CAGL0E06600g | 966    | Yes    | Yes   | Yes          |
  | QHS65618 | CAGL0F00099g | 4498   | No     | No    | Yes          |
  | QHS66001 | CAGL0F09251g | 1571   | No     | No    | Yes          |
  | QHS66441 | CAGL0H00209g | 1990   | No     | No    | Yes          |
  | QHS67215 | CAGL0I07293g | 1862   | Yes    | Yes   | No           |
  | QHS67369 | CAGL0I11000g | 2386   | No     | No    | Yes          |
  | QHS67886 | CAGL0J12067g | 2112   | No     | No    | Yes          |
  | QHS67887 | CAGL0K00110g | 833    | Yes    | No    | Yes          |
  | QHS68452 | CAGL0L00227g | 3242   | Yes    | Yes   | Yes          |
  | QHS69029 | CAGL0M00121g | 1581   | No     | No    | Yes          |

- To see if other genomes also contain a large number of missing homologs due to genome assembly issues, I downloaded a 2021 assembly for _S. cerevisiae_ S288C, the same as the genome reference strain. There is no publication associated with [this assembly](https://www.ncbi.nlm.nih.gov/assembly/GCA_016858165.1/). The webpage for it lists INSCRIPTA as the submitter. Sequencing was done using PacBio and the genome *is* annotated, allowing me to perform blastp. Using the same blastp queries and parameters, no hits were recovered. There are two possible explanations: the new PacBio assembly is still incomplete, especially with respect to the subtelomere region, or that it's _C. glabrata_ that has specifically expanded the Hil family in the subtelomere region.

- Not satisfied with the _S. cerevisiae_ result above, I identified another 2019 PacBio+Illumina [assembly](https://www.ncbi.nlm.nih.gov/assembly/GCA_007993695.1/) for _Kluyveromyces lactis_ CBS2105 (Varela, Wolfe et al 2019, PMID: 31813610). I could identify a single hit in the genome, consistent with the previous refseq result. So **it increasingly seems like _C. glabrata_ is an outlier.**

- Unclear how this family has evolved in the Nakaseomycetes, as the other genomes are not of the same quality as _C. glabrata_. Should we get involved in sequencing those species???

## 2022-05-03 [HB] Nakaseomyces blast, GRYC

_Background_

* Nakaseomyces and Lachancea genera blastp
* Decide whether to add FungiDB and GRYC hits
* Interpret _C. glabrata_ finding as an isolated instance vs general issue

_Notes_

* FungiDB doesn’t add any new species (all present in refseq), just different strains. I checked all previously “added” sequences from the FungiDB hit list and found them to all be included in the new refseq blast hit list.
* Nakaseomyces and Lachancea genera species assemblies are present in the Assembly database in NCBI but not in the refseq database. In addition, the assemblies that do exist are not annotated (no translated protein sequences). I thus downloaded the CDS translations from GRYC for the Nakaseomyces (except _C. glabrata_) and Lachancea (only included _L. kluyveri_ as a representative) genera and performed blastp searches locally.
    * I found a 2021 _C. nivariensis_ assembly for the same strain as the one available in GRYC (CBS 9983). The new assembly used Oxford Nanopore + iSeq and achieved >100x coverage. Performing tblastn searches in this new assembly led to one more hit than those recovered from the early assembly (3 vs 2).
* My plan is to add the GRYC sequences to the refseq list and use that as the basis for the new version.

## 2022-05-26 [HB] Expanded BLAST results summary

This is a summary of the past several days of analyses:

1. I performed blastp searches against the refseq database using three queries (Cgla_Hyr1, Calb_Hyr1, Caur_Hil1, just the PF11765 domain portion). 
   - In retrospect, I realized that what would be a better solution to this arbitrary choice of queries is to use the HMM model already existing for the PF11765 domain and perform a HMMER search. However, so far the online version of HMMER doesn't support searching against the refseq or any other ncbi databases. This is because HMMER, even though it has been significantly sped up in its recent versions, is still much slower compared with BLAST.
   - After combining the results from the three queries, I compared them to the previous 104 homologs list and found that among the species included in both, **only three additional hits will be added due to the addition of the two new queries** (two more hits are new but are below 500 aa, which had been excluded in the previous analysis). That's quite reassuring, suggesting that our initial (arbitrary) choice of Caur_Hil1 PF11765 domain didn't bias the hits towards species close to it. Note that using the Cgla_Hyr1 as a query didn't recover more hits from the Saccharomycetaceae family.
2. As described above, when compared with the new generation of assemblies that used long-read technologies, most of the hits do appear to be complete and match quite well. _C. tropicalis_, for which the refseq assemblies contain many incomplete proteins compared with the newer assembly, appears to be a special case.
3. When I performed an independent blastp search (instead of comparing the hits from the refseq hits to the new assembly) in the new _C. glabrata_ assembly generated by the Cormack lab, I could identify a larger set of hits (13 vs 2). I checked a few other species with newer assemblies, including _C. nivariensis_ (CBS9983), _S. cerevisiae_ (S288c), _K. lactis_ (CBS2105) and _S. stipitis_ (NRRL-Y7124). All had the same number or fewer hits (_S. stipitis_ had fewer hits).

## 2022-08-09 [HB] Spathaspora passalidarum blast

This is in response to reviewer 3's comment:

> The manuscript concludes that having more genes is better, that the gene family represents diversification that must be driven by its importance to pathogenesis, without recognizing that some species evolve toward lower pathogenesis. This concept could be explored in the Discussion. …My own experience makes me wonder if the authors found any examples of species that provide an exception to the idea that having more genes is better and positively associated with pathogenesis. The parallel between IFF/HYR and ALS genes is made many times in the manuscript. Spathaspora passalidarum, a species that is not pathogenic in humans, but clearly within the phylogenetic group examined here, has 29 loci with sequence similarity to ALS genes. How many IFF/HYR genes are in S. passalidarum?"

I found two assemblies for this species:

| Assembly        | GCA_013620965.1                                              | GCF_000223485.1 |
| --------------- | ------------------------------------------------------------ | --------------- |
| Strain          | NRRL Y-27907                                                 | NRRL Y-27907    |
| Date            | 2020/07/24                                                   | 2011/08/25      |
| Coverage        | 200.0x                                                       | 43.8x           |
| Sequencing      | Oxford nanopore + MiSeq                                      | 454; Sanger     |
| Assembly method | Albacore v1.2.4; Canu v. 1.5; Nanopolish v. 0.7.1; PILON v. 1.22 | Newbler v. 2.3  |
| Assembly level  | Contig                                                       | Scaffold        |
| # of seq        | 10                                                           | 8               |
| N50             | 1.75M                                                        | 2M              |

Note that the refseq assembly of this species is already part of the extended BLAST list. There we had 3 hits in this species in total and all passed the query coverage cutoff. I searched the newer assembly using tblastn and found 
