# Polypharmacy
Pharmacovigilance research project on polypharmacy-associated adverse drug reactions (ADRs) using FDA FAERS data. Includes SAS scripts, disproportionality analysis (ROR, PRR, IC, EBGM), and age-sex stratified signal detection.






I used this SAS CODE TO MAKE FILTERED FILES OF ALL THREE YEARS 2022,2023,2024.




/* Step 1: Assign Library */
libname faers '/home/u64263069/faers';

/* Step 2: Sort All Datasets by primaryid */
proc sort data=faers.polypharmacy_2024_all; by primaryid; run;
proc sort data=faers.demo2024_clean; by primaryid; run;
proc sort data=faers.outc2024_clean; by primaryid; run;
proc sort data=faers.reac2024_clean; by primaryid; run;

/* Step 3: Merge All Datasets by primaryid */
data work.polypharmacy_2024_merged;
    merge faers.polypharmacy_2024_all (in=drug)
          faers.demo2024_clean (in=demo)
          faers.outc2024_clean (in=outc)
          faers.reac2024_clean (in=reac);
    by primaryid;
    run;


/* Step 4: Clean Missing Critical Data */
data work.polypharmacy_2024_clean;
    set work.polypharmacy_2024_merged;

    /* Remove missing primaryid or drugname */
    if primaryid = . then delete;
    if missing(drugname) then delete;

    /* Remove missing adverse event terms (pt) */
    if missing(pt) then delete;

    /* Standardize text variables */
    drugname = upcase(drugname);
    prod_ai  = upcase(prod_ai);
    role_cod = upcase(role_cod);
    pt       = upcase(pt);

    /* Handle missing sex and age */
    if missing(sex) then sex = 'UNK';
    if missing(age) then age = -1;

    /* Handle missing dose_form */
    if missing(dose_form) then dose_form = 'UNKNOWN';
run;

/* Step 5: Filter Age > 65 */
data work.polypharmacy_statin24_allfile;
    set work.polypharmacy_2024_clean;
    where age > 65;
run;

data faers.polypharmacy_statin24_allfile;
    set work.polypharmacy_statin24_allfile;
run;
