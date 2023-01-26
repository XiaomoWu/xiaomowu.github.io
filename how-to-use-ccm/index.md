# The correct way to use CCM


CCM is a product provided by CRSP to link the Compustat database (`gvkey`) to CRSP (`permno`). In CCM, there're three tables you can use to make the link:

- `ccmxpf_linktable`
- `ccmxpf_lnkhist`
- `ccmxpf_lnkused`

Which one should you use?

# TL;DR
- WRDS recommend using `ccmxpf_lnkhist`:

{{< admonition type=quote >}}
*"The link history table (CCMXPF_LNKHIST) is **the main table used for WRDS CCM web queries**. It provides **access to all Compustat records**, regardless of whether or not the securities are in the CRSP universe."*
{{< /admonition >}}

- **`ccmxpf_lnktable` is deprecated and should not be used.** 

  Before Feb 2014, `ccmxpf_lnktable` is used to add `usedflag` to `ccmxpf_lnkused`. But after Feb 2014 `usedflag` is no longer needed, hence `ccmxpf_lnktable` become meaningless:

{{< admonition type=quote >}}
*"SAS programmers will also notice a change.  Since Used Flag is no longer needed the dataset CCMXPF_LINKTABLE is also not needed.  This table added Used Flag from CCMXPF_LNKUSED to the data in CCMXPF_LNKHIST.  The Link History table is "Compustat-centric" - it has all companies, whether or not a link exists to a security in CRSP.  It has all the companies in the Link Used table and more. **We recommend using the Link History dataset** directly."*
{{< /admonition >}}

- `ccmxpf_lnkused` is safe to use, but it's CRSP centric, meaning it has a smaller stock universe than `ccmxpf_lnkhist`. For most of the time, you should use `ccmxpf_lnkhist` instead of `ccmxpf_lnkused.`

{{< admonition type=quote >}}
*"The link used table (CCMXPF_LNKUSED) has **all records of GVKEYs with a match to a** **`PERMNO`** **identifier**, even those not used. This table is **CRSP-centric** (i.e. starting with `PERMNO`), compared to the link history table which is Compustat-centric (i.e. starting with `GVKEY`). This table allows users searching by `PERMNO` to find all possible `GVKEYs` that have ever been linked, including those not used during a certain period."*
{{< /admonition >}}

# Clean `ccmxpf_lnkhist`

While `lnkhist` is the go-to table, it has an issue of "consecutive observations." As an illustration, see the following example:
```R
  gvkey lpermno lpermco linkdt     linkenddt 
1 001010 10006  22156   1950-05-01 1962-01-30 
2 001010 10006  22156   1962-01-31 1984-06-28
```

As you can see, line 1 and 2 should be merged into a single line:
```R
  gvkey lpermno lpermco linkdt     linkenddt 
1 001010 10006  22156   1950-05-01 1981-06-28 
```

WRDS provides an [SAS example](https://wrds-www.wharton.upenn.edu/pages/wrds-research/macros/wrds-macros-ccm_lnktablesas/) to "collapse" the consecutive observations (yeah I know the question "who's still using SAS" is like "who's still buying GTA5"). I translate the SAS code to R
```R
library(data.table)

# get the "ccmxpf_lnkhist" table
# make sure ccm is a data.table
ccm = get_ccmxpf_lnkhist()

# create previous linkenddt and linkdt
ccm = ccm[, ':='(last_linkdt=shift(linkdt), last_linkenddt=shift(linkenddt)),
	  keyby=.(gvkey, lpermno)]

# merge consecutive observations
ccm = ccm[
    # if linkdt is on the same day or 1 day after last_linkenddt, 
    # rewrite linkdt to last_linkdt
      linkdt==last_linkenddt+1 | linkdt==last_linkenddt,   
      ':='(linkdt=last_linkdt)
    # only keep the last observation of (gvkey, lpermno, linkdt) pair
    ][order(gvkey, lpermno, linkdt, linkenddt)
    ][, .SD[.N], keyby=.(gvkey, lpermno, linkdt)
    ][, c('last_linkdt', 'last_linkenddt') := NULL]
```

