CrossDomainReviewTextRecommendation
===================================

.. contents:: 目录

Review-Aware Cross Domain Production Recommendation
---------------------------------------------------
Focus on Rating Prediction Problem

* Incorporating the sentiment information of review text into the modeling of user.
* Transform the User Domain-Shared Preference from source domain to target domain by DSN [#]_ for cold start user.
* Generate the rating for cold start user and product in target domain by Matrix Factorization.

All the experiments is based on Amazon Dataset.
You could download them at http://jmcauley.ucsd.edu/data/amazon/

Good Luck

Specification of Path Tree
--------------------------

::


  ├── CDRTR
  │   ├── core
  │   │   ├── DeepModel: code path of some DL model, including SentiRec, DSN, DSNRec
  │   │   └── LinearModel: code path of some linear model, including LR
  │   ├── dataset: dataset adapter for SentiRec, DSNRec
  │   └── preprocess: The process of preprocess, including generating the vocabulary of two domain
  │                   Word Embedding, tool to count the number of cool start user
  ├── exam: directory containes the experiment dataset, script to generate the rmse result
  │   ├── data: small dataset to test the code
  │   │   ├── preprocess: path to save the output of some the preprocess
  │   │   └── source: path to save source dataset
  │   ├── log
  │   │   ├── baseline
  │   │   ├── debug_generateVoca_MultiCross
  │   │   ├── debug_mergeUI_MultiCross
  │   │   ├── DSNRec
  │   │   └── sentitrain
  │   └── MultiCross: path the dataset to explort the cross-available of domains
  │       ├── Beauty_Clothin_Shoes_and_Jewelry
  │       ├── Beauty_Movies_and_TV
  │       ├── Kindle_Beauty
  │       ├── Kindle_Clothin_Shoes_and_Jewelry
  │       ├── Kindle_Movies_and_TV
  │       └── Movies_and_TV_Clothin_Shoes_and_Jewelry
  └── tests: some test code

How to Run it?
---------------
There is a makefile under the exam directory.
The content of it is shown as below.

::

    export PYTHONPATH=..

    transCSV:
        python -m CDRTR.preprocess csv_format --mode $(MODE) --fields reviewerID,asin,overall --data_dir $(DATA)

    # make generatevoca MODE=DEBUG DATA=data
    generatevoca:
        python -m CDRTR.preprocess generatevoca --mode $(MODE) --fields asin,reviewerID,overall,reviewText,unixReviewTime --data_dir $(DATA)

    # make extractinfo MODE=DEBUG
    extractinfo:
        python -m CDRTR.preprocess extractinfo --mode $(MODE)

    preprocess: generatevoca extractinfo

    # make sentitrain DATA=data DOMAIN=Music
    # make sentitrain DATA=data DOMAIN=Auto
    sentitrain:
        python -m CDRTR.core.DeepModel.SentiRec --dir $(DATA)/preprocess --domain $(DOMAIN) --filter_size 3,5,7,11 --epoches 400

    mergeUI:
        python -m CDRTR.preprocess mergeui --mode $(MODE) --data_dir $(DATA)

    DSNRec:
        python -m CDRTR.core.DeepModel.DSNRec --data_dir $(DATA) --src_domain $(SRCDO) --tgt_domain $(TGTDO) --epoches $(EPOCH) --mode $(MODE)


1. generatevoca: transform the json data saved in data/source/ to some file, including the vocabulary of two domain, statistic of cold start user and so on.

2. transCSV: transform json data into csv file, which is used by MyMediaLite to perform the baseline.

3. sentitrain: to train sentiRec [#]_ with dataset, taking the output of the lastest layer of CNN as the vector form of review text.

4. mergeUI: merge the vector form of review text into user/product representation.

5. DSNRec: DSNRec to run on specific dataset.

the pipeline.sh file under exam/ directory containes the execution sequence of makefile.

::

    Usage pipeline.sh data_dir src_domain tgt_domain epoch

example:

::

  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam$ ./pipeline.sh data Auto Musi 400
  ...
  ...
  # after you run DSNRec on several dataset, you can output the RMSE by the script below.
  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam$ python show_result.py `find . -name "test*.pk" | grep -v MultiCross`
  +-------------+----------------+
  |    domain   |      rmse      |
  +-------------+----------------+
  |     Cell    | 1.02770589547  |
  |   Digital   |  0.9646660533  |
  | Electronics | 1.09582470097  |
  |    Kindle   | 1.01310818718  |
  |     Musi    | 0.737983482096 |
  |    Tools    | 0.94334647007  |
  |     CDs     | 1.00803053039  |
  |    Video    | 0.987097572114 |
  |   Jewelry   | 1.04308169038  |
  |    Movie    | 1.00178416516  |
  |    Beauty   | 1.09342681075  |
  |    Sports   | 0.901738399564 |
  |     Auto    | 0.937711151504 |
  |    Office   | 0.855115969443 |
  |     Toys    | 0.911580666733 |
  +-------------+----------------+

  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam$ python show_user_record_count.py `ls -F | grep / | grep -v log | grep -v MultiCross | cut -d/ -f1`
  +----------------------------------------------+--------------+--------------+--------------+------------------+-----------------+
  |                   domains                    | srcUserCount | tgtUserCount | overlapCount |  srcOverlapRate  |  tgtOverlapRate |
  +----------------------------------------------+--------------+--------------+--------------+------------------+-----------------+
  |      Beauty/Clothing_Shoes_and_Jewelry       |    18143     |    35167     |     4220     |  0.232596593728  |  0.11999886257  |
  |          CDs_and_Vinyl/Electronics           |    68998     |    186143    |     6260     |  0.090727267457  | 0.0336300586109 |
  |   Cell_Phones_and_Accessories/Video_Games    |    26480     |    22904     |     1399     |  0.052832326284  | 0.0610810338805 |
  |         Toys_and_Games/Digital_Music         |    19233     |     5362     |     179      | 0.00930692039723 | 0.0333830660201 |
  | Sports_and_Outdoors/Grocery_and_Gourmet_Food |    33563     |    12646     |     2035     | 0.0606322438399  |  0.160920449154 |
  |          Kindle_Store/Movies_and_TV          |    65469     |    121206    |     2754     | 0.0420657104889  | 0.0227216474432 |
  |  Office_Products/Tools_and_Home_Improvement  |     3012     |    14745     |     1893     |  0.628486055777  |  0.128382502543 |
  +----------------------------------------------+--------------+--------------+--------------+------------------+-----------------+

  +----------------------------------------------+------------+---------------+------------+---------------+
  |                   domains                    | srcURcount | srcOverRcount | tgtURcount | tgtOverRcount |
  +----------------------------------------------+------------+---------------+------------+---------------+
  |      Beauty/Clothing_Shoes_and_Jewelry       |   149091   |     49411     |   238420   |     40257     |
  |          CDs_and_Vinyl/Electronics           |   968881   |     128711    |  1589951   |     99237     |
  |   Cell_Phones_and_Accessories/Video_Games    |   179952   |     14487     |   211931   |     19849     |
  |        Automotive/Musical_Instruments        |   20117    |      356      |    9842    |      419      |
  |         Toys_and_Games/Digital_Music         |   164876   |      2721     |   61975    |      2731     |
  | Sports_and_Outdoors/Grocery_and_Gourmet_Food |   273513   |     22824     |   114343   |     36911     |
  |          Kindle_Store/Movies_and_TV          |   932521   |     50098     |  1628754   |     68779     |
  |  Office_Products/Tools_and_Home_Improvement  |   26170    |     27088     |   111009   |     23467     |
  +----------------------------------------------+------------+---------------+------------+---------------+
  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam$ python baseline_result_show.py
  +----------------------------+-------------------------------+---------------------------+----------+-------------+---------+-----------------+---------------+-------------+-----------------------------+
  |           domain           | FactorWiseMatrixFactorization | BiasedMatrixFactorization | SlopeOne | SVDPlusPlus |  Random | BiPolarSlopeOne | GlobalAverage | ItemAverage | LatentFeatureLogLinearModel |
  +----------------------------+-------------------------------+---------------------------+----------+-------------+---------+-----------------+---------------+-------------+-----------------------------+
  |            Musi            |            0.74917            |          0.75819          | 0.74003  |    0.7429   | 2.07642 |     0.74003     |    0.74003    |   0.79938   |            2.5739           |
  |       Toys_and_Games       |            0.91575            |          0.90545          | 1.01175  |   0.94514   | 1.96148 |     1.01175     |    1.01175    |   0.91091   |           2.22349           |
  | Tools_and_Home_Improvement |            0.92429            |          0.93079          | 0.94901  |   0.93271   | 2.02191 |     0.94901     |    0.94901    |   0.98844   |           3.01554           |
  |          Jewelry           |            1.07836            |          1.08417          | 1.09712  |   1.08301   | 2.02008 |     1.09712     |    1.09712    |   1.12147   |           2.63191           |
  |           Beauty           |             1.1154            |          1.12241          | 1.14271  |   1.11946   | 2.01473 |     1.14271     |    1.14271    |   1.16054   |           2.46513           |
  |      Office_Products       |            0.86723            |          0.86452          |   0.91   |   0.88177   | 1.99483 |       0.91      |      0.91     |   0.89912   |           2.39326           |
  |            Auto            |            0.91939            |          0.91734          | 0.95858  |   0.93183   | 2.06752 |     0.95858     |    0.95858    |   0.95209   |           2.65961           |
  |       CDs_and_Vinyl        |            0.95356            |          0.95376          | 1.00893  |   0.96628   | 2.02005 |     1.00893     |    1.00893    |   0.96833   |           2.60192           |
  |        Cell_Phones         |            1.04561            |          1.05164          | 1.09231  |   1.05525   | 1.98757 |     1.09231     |    1.09231    |   1.06846   |           2.19888           |
  |          Grocery           |            0.98135            |          0.98238          |  1.0676  |   0.99329   | 1.94582 |      1.0676     |     1.0676    |   0.99631   |           2.15443           |
  |           Sports           |            0.90509            |          0.90759          | 0.93544  |    0.9153   |  2.0045 |     0.93544     |    0.93544    |   0.94513   |           2.72289           |
  |        Video_Games         |            1.00654            |          1.00706          | 1.08674  |   1.02059   | 1.96103 |     1.08674     |    1.08674    |   1.02006   |           1.98473           |
  |       Digital_Music        |            0.99705            |          0.98776          | 1.08185  |   1.01934   | 1.97827 |     1.08185     |    1.08185    |   0.99791   |           2.34118           |
  |           Kindle           |            0.99793            |           0.9961          | 1.04402  |   1.01122   | 1.96238 |     1.04402     |    1.04402    |    1.0107   |           2.61299           |
  |        Electronics         |            1.07022            |          1.07283          | 1.12268  |   1.08046   | 2.01596 |     1.12268     |    1.12268    |   1.08378   |           1.91997           |
  |       Movies_and_TV        |            0.97773            |          0.96989          | 1.09381  |   0.99628   | 1.96544 |     1.09381     |    1.09381    |   0.96867   |           1.73305           |
  +----------------------------+-------------------------------+---------------------------+----------+-------------+---------+-----------------+---------------+-------------+-----------------------------+


Checking the structure of DSNRec by Tensorboard
-----------------------------------------------

::

  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam$ cd log/DSNRec/Auto_Musi/
  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam/log/DSNRec/Auto_Musi$ ls
  test_mses.pk  train  Untitled.ipynb
  winchua@CCNLForDL:~/CrossDomainReviewTextRecommendation/CDRTR/exam/log/DSNRec/Auto_Musi$ tensorboard --logdir=train --port 12345


.. image:: https://raw.githubusercontent.com/WinChua/CDRTR/master/docs/source/_static/model.bmp

Reference
---------

.. [#] Bousmalis, K., Trigeorgis, G., Silberman, N., Krishnan, D., & Erhan, D. (2016). Domain SeparationNetworks, (Nips). Retrieved from http://arxiv.org/abs/1608.06019
.. [#] Hyun, D., Park, C., Yang, M.-C., Song, I., Lee, J.-T., & Yu, H. (2018). Review Sentiment-Guided Scalable DeepRecom-mender System. Ann SIGIR, 18, 965–968. https://doi.org/10.1145/3209978.3210111



✨🍰✨
