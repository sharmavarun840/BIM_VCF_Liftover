# bim-vcf-liftover

A single file, browser based tool for lifting PLINK `.bim` files and VCF / VCF.gz files between human genome builds (hg18, hg19, hg38). Everything runs client side in JavaScript. No data is uploaded anywhere.

Live logic only, no backend: open `bim-liftover.html` in any modern browser and it works entirely offline once the page and its one CDN dependency (pako, for gzip) have loaded.

## Features

- Supports all four practical conversions: hg18 to hg19, hg18 to hg38, hg19 to hg38, hg38 to hg19
- Accepts standard UCSC `.chain` or `.chain.gz` files
- Auto detects `.bim` vs `.vcf` / `.vcf.gz` input, by filename or by sniffing content
- Handles gzip by magic bytes, not just file extension, so renamed or mislabeled files still work
- Chunked, non blocking processing with a progress bar, so large files do not freeze the tab
- Per chromosome mapping stats and a preview of the first lifted rows
- Clear, itemized reasons for every unmapped variant (gap, deleted region, alternate contig, chromosome not in chain, and so on)

### VCF specific

- Reverse complements REF and ALT when a variant lands on a reverse strand chain block
- Checks that multi base REF alleles (indels) do not span an alignment block boundary before lifting them
- Detects and excludes structural variants and breakend records (symbolic ALT like `<DEL>`, or ALT containing `[` / `]`), since these need dedicated interval logic this tool does not implement
- Drops stale `##contig` header lines and inserts a `##liftover=...` provenance line
- Re-sorts output by chromosome and position
- Outputs both a plain `.vcf` and a gzip compressed `.vcf.gz`

### PLINK specific

- Outputs a lifted `.bim`, plus an unmapped report
- Also outputs a PLINK ready exclude list and keep list (one variant ID per line), since a lifted `.bim` will usually have fewer rows than the original, and the corresponding `.bed` / `.fam` need to be filtered with `plink --exclude` or `plink --extract` to stay in sync before you swap in the new positions

## Usage

1. Open `bim-liftover.html` in a browser.
2. Pick a source to target build conversion.
3. Download the matching chain file from UCSC using the link shown, and drop it into the chain file box.
4. Drop in your `.bim`, `.vcf`, or `.vcf.gz` file.
5. Click **Run liftover**.
6. Download the lifted output, plus the unmapped report and summary.

## Algorithm

Full details are in [`liftover-algorithm.md`](./liftover-algorithm.md). In short:

1. Parse UCSC chain blocks into sorted, per chromosome intervals of `{tStart, tEnd, qStart, qStrand, qSize, score}`.
2. For each input coordinate, binary search the interval containing it, then apply a linear offset, with a strand flip formula (`qSize - (qStart + offset) - 1`) when the block is on the reverse strand.
3. For VCF, additionally reverse complement REF/ALT on strand flips, verify indels do not cross a block boundary, and exclude structural variants and breakends rather than attempt to lift them.

This mirrors the core lookup logic of UCSC's own `liftOver` for simple point and span features.

## Known limitations

- REF alleles are not verified against the target build's actual reference FASTA (this tool has no access to one). For production pipelines, spot check results against a proper reference.
- Structural variants, breakends, and any feature that cannot be expressed as a point or a single contained span are not lifted.
- The `.vcf.gz` output is standard gzip, not BGZF. It is readable directly by bcftools and vcftools, but re-compress with `bgzip` if you need tabix indexing.
- Overlapping chains on the same chromosome are resolved with a simple higher score wins rule, not a full realignment.

## Chain files

Chain files are not bundled with this tool. Download them directly from UCSC:

- [hg18ToHg19.over.chain.gz](https://hgdownload.soe.ucsc.edu/goldenPath/hg18/liftOver/hg18ToHg19.over.chain.gz)
- [hg18ToHg38.over.chain.gz](https://hgdownload.soe.ucsc.edu/goldenPath/hg18/liftOver/hg18ToHg38.over.chain.gz)
- [hg19ToHg38.over.chain.gz](https://hgdownload.soe.ucsc.edu/goldenPath/hg19/liftOver/hg19ToHg38.over.chain.gz)
- [hg38ToHg19.over.chain.gz](https://hgdownload.soe.ucsc.edu/goldenPath/hg38/liftOver/hg38ToHg19.over.chain.gz)

These files are property of The Regents of the University of California, made available for non commercial research use. See UCSC's download page for full terms.

## License

Add your preferred license here (for example MIT). This repository does not redistribute any UCSC chain file data.
