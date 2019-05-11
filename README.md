# utl-drop-all-columns-in-all-tables-where-the-column-names-contains-specified-text
Drop all columns in all tables where the column names contains specified text
    Drop all columns in all tables where the column names contains specified text

    Here we drop all columns where the column name contains "SEPAL"

    github
    https://tinyurl.com/y33mr53a
    https://github.com/rogerjdeangelis/utl-drop-all-columns-in-all-tables-where-the-column-names-contains-specified-text

    SAS Forum
    https://tinyurl.com/y2tvte9n
    https://communities.sas.com/t5/SAS-Programming/Go-through-all-datasets-in-a-directory-and-delete-columns-that/m-p/557920/highlight/true#M15

    The solution below produces a log and will not corrupt the table if there is an error.

    Due to performance issues on large EG servers the code below may take to long to be useful.
    Should be fine on recent local laptops or desktop workstations.
    Proc sql dictionaries are too slow on 'non-programmer" large EG clusters.

    *_                   _
    (_)_ __  _ __  _   _| |_
    | | '_ \| '_ \| | | | __|
    | | | | | |_) | |_| | |_
    |_|_| |_| .__/ \__,_|\__|
            |_|
    ;

    options validvarname=upcase;

    proc datasets lib=work kill;
    run;quit;;

    data havea
         haveb (keep=species sepalwidth)
         havec (drop=sepal:);
     set sashelp.iris(obs=3);

    run;quit;


    Up to 40 obs from WORK.HAVEA total obs=3

                          REMOVE THESE
                      =========================
    Obs    SPECIES    SEPALLENGTH    SEPALWIDTH    PETALLENGTH    PETALWIDTH

     1     Setosa          50            33             14             2
     2     Setosa          46            34             14             3
     3     Setosa          46            36             10             2


    Up to 40 obs from WORK.HAVEB total obs=3

                        REMOVE
                      ==========
    Obs    SPECIES    SEPALWIDTH

     1     Setosa         33
     2     Setosa         34
     3     Setosa         36


     Up to 40 obs from WORK.HAVEC total obs=3

    DO NOTHING
    ==========

     Obs    SPECIES    PETALLENGTH    PETALWIDTH

      1     Setosa          14             2
      2     Setosa          14             3
      3     Setosa          10             2

    *            _               _
      ___  _   _| |_ _ __  _   _| |_
     / _ \| | | | __| '_ \| | | | __|
    | (_) | |_| | |_| |_) | |_| | |_
     \___/ \__,_|\__| .__/ \__,_|\__|
                    |_|
    ;

    Note HAVEC is untouched because HAVEC has no column names with "SEPAL"

      WORK.LOG total obs=2

      TABLE   RC  VARLST                                          STATUS

      HAVEA    0  SEPALWIDTH SEPALLENGTH    Variables SEPALWIDTH SEPALLENGTH dropped from HAVEA
      HAVEB    0  SEPALWIDTH                Variables SEPALWIDTH dropped from HAVEB


      WORK.HAVEA total obs=3

      Obs    SPECIES    PETALLENGTH    PETALWIDTH

       1     Setosa          14             2
       2     Setosa          14             3
       3     Setosa          10             2

      HAVEB total obs=3

      Obs    SPECIES

       1     Setosa
       2     Setosa
       3     Setosa

    *
     _ __  _ __ ___   ___ ___  ___ ___
    | '_ \| '__/ _ \ / __/ _ \/ __/ __|
    | |_) | | | (_) | (_|  __/\__ \__ \
    | .__/|_|  \___/ \___\___||___/___/
    |_|
    ;

    * make sure the enviroment is cleared;
    %symdel table variable varLst cc / nowarn;

    proc datasets lib=work nolist;
     delete _tmp_ log;
    run;quit;

    * make data;
    data havea
         haveb (keep=species sepalwidth)
         havec (drop=sepal:);
     set sashelp.iris(obs=3);

    run;quit;

    proc sql;
      create
         table meta as
      select
         memname as table
        ,name    as variable
      from
        (
         select
            memname
           ,name
         from
           sashelp.vcolumn
         group
           by memname
         having
           libname="WORK" and
           max(index(name,"SEPAL")) ge 1
        )
      where
         name eqt "SEPAL"
    ;quit;

    /*

    Up to 40 obs WORK.META total obs=3

    Obs    TABLE     VARIABLE

     1     HAVEA    SEPALLENGTH
     2     HAVEA    SEPALWIDTH
     3     HAVEB    SEPALWIDTH
    */


    data log(drop=variable);

       * create a table with just the meta dataneeded;

       if _n_=0 then do; %let rc=%sysfunc(dosubl('

          proc sql;
            create
               table meta as
            select
               memname as table
              ,name    as variable
            from
              (
               select
                  memname
                 ,name
               from
                 sashelp.vcolumn
               group
                 by memname
               having
                 libname="WORK" and
                 max(index(name,"SEPAL")) ge 1
              )
            where
               name eqt "SEPAL"
          ;quit;
          '));

          /* Note HAVEC is not listed because it does not have a column with "SEPAL"
          WORK.META

          TABLE     VARIABLE

          HAVEA    SEPALLENGTH
          HAVEA    SEPALWIDTH
          HAVEB    SEPALWIDTH
          */

       end;

       retain varLst ;
       length varLst status $200;

       set meta;
       by table;

       varLst=catx(" ",varLst,variable);

       if last.table then do;

         call symputx("varLst",varLst);
         call symputx("table",table);

         rc=dosubl('
            data _tmp_;
               set &table.(drop=&varLst);
            run;quit;
            %let cc=&syserr;
         ');

         if symgetn("cc")=0 then do;

            rc=dosubl('

              proc sql;
                 drop table &table
              ;quit;

              proc datasets lib=work nolist;
                 change _tmp_= &table;
              run;quit;
           ');

           status   =catx(" ","Variables", varLst, "dropped from", table );

         end;

         else status=catx(" ","ERROR: Variables", varLst, "NOT dropped from", table, "table unchanged" );

         output;

         varLst="";

      end;


    run;quit;

    *_
    | | ___   __ _
    | |/ _ \ / _` |
    | | (_) | (_| |
    |_|\___/ \__, |
             |___/
    ;

    NOTE: The file WORK._TMP_ (memtype=DATA) was not found, but appears on a DELETE statement.
    NOTE: Deleting WORK.LOG (memtype=DATA).
    2226!     quit;

    NOTE: PROCEDURE DATASETS used (Total process time):
          real time           0.02 seconds
          user cpu time       0.00 seconds
          system cpu time     0.01 seconds
          memory              350.62k
          OS Memory           26100.00k
          Timestamp           05/11/2019 06:49:55 PM
          Step Count                        414  Switch Count  0


    2227  * make data;
    2228  data havea
    2229       haveb (keep=species sepalwidth)
    2230       havec (drop=sepal:);
    2231   set sashelp.iris(obs=3);
    2232  run;

    NOTE: There were 3 observations read from the data set SASHELP.IRIS.
    NOTE: The data set WORK.HAVEA has 3 observations and 5 variables.
    NOTE: The data set WORK.HAVEB has 3 observations and 2 variables.
    NOTE: The data set WORK.HAVEC has 3 observations and 3 variables.


    2232!     quit;
    2233  proc sql;
    2234    create
    2235       table meta as
    2236    select
    2237       memname as table
    2238      ,name    as variable
    2239    from
    2240      (
    2241       select
    2242          memname
    2243         ,name
    2244       from
    2245         sashelp.vcolumn
    2246       group
    2247         by memname
    2248       having
    2249         libname="WORK" and
    2250         max(index(name,"SEPAL")) ge 1
    2251      )
    2252    where
    2253       name eqt "SEPAL"
    2254  ;
    NOTE: The query requires remerging summary statistics back with the original data.
    NOTE: Table WORK.META created, with 3 rows and 2 columns.

    2254!  quit;
    NOTE: PROCEDURE SQL used (Total process time):

    2255  /*
    2256  Up to 40 obs WORK.META total obs=3
    2257  Obs    TABLE     VARIABLE
    2258   1     HAVEA    SEPALLENGTH
    2259   2     HAVEA    SEPALWIDTH
    2260   3     HAVEB    SEPALWIDTH
    2261  */
    2262  data log(drop=variable);
    2263     * create a table with just the meta dataneeded;
    2264     if _n_=0 then do; %let rc=%sysfunc(dosubl('
    2265        proc sql;
    2266          create
    2267             table meta as
    2268          select
    2269             memname as table
    2270            ,name    as variable
    2271          from
    2272            (
    2273             select
    2274                memname
    2275               ,name
    2276             from
    2277               sashelp.vcolumn
    2278             group
    2279               by memname
    2280             having
    2281               libname="WORK" and
    2282               max(index(name,"SEPAL")) ge 1
    NOTE: The query requires remerging summary statistics back with the original data.
    NOTE: Table WORK.META created, with 3 rows and 2 columns.

    NOTE: PROCEDURE SQL used (Total process time):

    2283            )
    2284          where
    2285             name eqt "SEPAL"
    2286        ;quit;
    2287        '));
    2288        /* Note HAVEC is not listed because it does not have a column with "SEPAL"
    2289        WORK.META
    2290        TABLE     VARIABLE
    2291        HAVEA    SEPALLENGTH
    2292        HAVEA    SEPALWIDTH
    2293        HAVEB    SEPALWIDTH
    2294        */
    2295     end;
    2296     retain varLst ;
    2297     length varLst status $200;
    2298     set meta;
    2299     by table;
    2300     varLst=catx(" ",varLst,variable);
    2301     if last.table then do;
    2302       call symputx("varLst",varLst);
    2303       call symputx("table",table);
    2304       rc=dosubl('
    2305          data _tmp_;
    2306             set &table.(drop=&varLst);
    2307          run;quit;
    2308          %let cc=&syserr;
    2309       ');
    2310       if symgetn("cc")=0 then do;
    2311          rc=dosubl('
    2312            proc sql;
    2313               drop table &table
    2314            ;quit;
    2315            proc datasets lib=work;
    2316               change _tmp_= &table;
    2317            run;quit;
    2318         ');
    2319         status   =catx(" ","Variables", varLst, "dropped from", table );
    2320       end;
    2321       else status=catx(" ","ERROR: Variables", varLst, "NOT dropped from", table, "table unchanged" );
    2322       output;
    2323       varLst="";
    2324    end;
    2325  run;

    SYMBOLGEN:  Macro variable TABLE resolves to HAVEA
    SYMBOLGEN:  Macro variable VARLST resolves to SEPALWIDTH SEPALLENGTH
    NOTE: There were 3 observations read from the data set WORK.HAVEA.
    NOTE: The data set WORK._TMP_ has 3 observations and 3 variables.
    NOTE: DATA statement used (Total process time):


    SYMBOLGEN:  Macro variable SYSERR resolves to 0
    SYMBOLGEN:  Macro variable TABLE resolves to HAVEA
    NOTE: Table WORK.HAVEA has been dropped.
    SYMBOLGEN:  Macro variable TABLE resolves to HAVEA
    NOTE: Changing the name WORK._TMP_ to WORK.HAVEA (memtype=DATA).
    NOTE: PROCEDURE DATASETS used (Total process time):


    SYMBOLGEN:  Macro variable TABLE resolves to HAVEB
    SYMBOLGEN:  Macro variable VARLST resolves to SEPALWIDTH
    NOTE: There were 3 observations read from the data set WORK.HAVEB.
    NOTE: The data set WORK._TMP_ has 3 observations and 1 variables.
    NOTE: DATA statement used (Total process time):


    SYMBOLGEN:  Macro variable SYSERR resolves to 0
    SYMBOLGEN:  Macro variable TABLE resolves to HAVEB
    NOTE: Table WORK.HAVEB has been dropped.
    NOTE: PROCEDURE SQL used (Total process time):

    SYMBOLGEN:  Macro variable TABLE resolves to HAVEB
    NOTE: Changing the name WORK._TMP_ to WORK.HAVEB (memtype=DATA).
    NOTE: PROCEDURE DATASETS used (Total process time):


    NOTE: There were 3 observations read from the data set WORK.META.
    NOTE: The data set WORK.LOG has 2 observations and 4 variables.
    NOTE: DATA statement used (Total process time):

    2325!     quit;

