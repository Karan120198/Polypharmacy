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






MERGING AND SORTING AND FILTERING FOR FINAL RESULTS 



libname faers "/home/u64263069/faers"

data faers.all_polypharmacy;
    set faers.polypharmacy_statin_merged
        faers.polypharmacy_hyperten_merged
        faers.polypharmacy_diabetes_merged;
run;


data faers.all_clean;
    set faers.all_polypharmacy;
    age_years = input(age, 8.);
    sex = upcase(sex);
    if sex in ('M', 'F'); /* Keep only known */
    if serious in (1, 2, 3, 4, 5, 6) then sae_flag = 1;
    else sae_flag = 0;
run;

data faers.flagged;
    set faers.all_clean;
    length drug_class $20;
    if index(drugname, 'ATORVASTATIN') > 0 or index(drugname, 'ROSUVASTATIN') > 0 then drug_class = 'Statin';
    else if index(drugname, 'LISINOPRIL') > 0 or index(drugname, 'AMLODIPINE') > 0 then drug_class = 'Hypertension';
    else if index(drugname, 'METFORMIN') > 0 or index(drugname, 'EMPAGLIFLOZIN') > 0 then drug_class = 'Diabetes';
run;

data faers.flagged;
    set faers.flagged;

    age_years = input(age, 8.);

    /* Age groups only for seniors */
    if 65 <= age_years < 75 then age_group = '65–74';
    else if 75 <= age_years < 85 then age_group = '75–84';
    else if age_years >= 85 then age_group = '85+';
    else age_group = 'Unknown';

run;



proc freq data=faers.flagged;
    tables drug_class*sex*sae_flag / chisq nocol nopercent;
    tables drug_class*age_group*sae_flag / chisq;
run;




proc sql;
    create table faers.poly_flags as
    select primaryid,
           max(drug_class = "Statin") as has_statin,
           max(drug_class = "Hypertension") as has_hyperten,
           max(drug_class = "Diabetes") as has_diabetes
    from faers.flagged
    group by primaryid;
quit;


data faers.therapy_groups;
    set faers.poly_flags;
    length therapy_group $20;
    total_classes = has_statin + has_hyperten + has_diabetes;

    if total_classes = 1 then therapy_group = "Monotherapy";
    else if total_classes = 2 then therapy_group = "Dual Therapy";
    else if total_classes = 3 then therapy_group = "Triple Therapy";
run;

proc sql;
    create table faers.flagged_poly as
    select a.*, b.therapy_group
    from faers.flagged as a
    left join faers.therapy_groups as b
    on a.primaryid = b.primaryid;
quit;


proc freq data=faers.flagged_poly;
    tables PT / missing;
run;


proc freq data=faers.flagged_poly noprint;
    tables therapy_group*drug_class*PT / out=faers.adr_counts_poly;
run;

proc sort data=faers.adr_counts_poly;
    by therapy_group drug_class descending count;
run;

proc print data=faers.adr_counts_poly (obs=100);
    where count >= 5;
run;



