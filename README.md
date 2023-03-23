# AnalysisChain
Full analysis chain starting from microAOD production (from miniAOD) to limit plots

The miniAOD datasets are centrally produced. They have different parameters associated with them. Currently, I can remember the following terms that I don't fully understand. Campaign name(?), Eras(?), MetaConditions(?), ReReco(?), UL (ultra legacy)(?), EOY(end of year)(?), Global Tag. When ID value is upgraded only the cut value is changed in case of cut-based and weight file in case of MVA ID. MVAestimatorV2(?)

V2 IDs are absent, only V1 IDs are present. PostRecoTools were recommended to created V2 IDs for the microAOD. Scales and smearings (?).

We do not use flashgg for SM higgs. Our main analyzer is diPhoMoriond_2017.py. beside flashgg we also have MonoHiggs where the analysis is done. microAOD is used to produce diphoton objects from falshgg. MicroAOD is used to analyze the diphoton objects. CMSSW analyzer written by us. After ntuplizer standard C scripts can be run on it. For analysis then ROOT is enough. MiniAOD uses EDM data format and hence, flashgg libraries are required. Usage of flashgg means that it is being used. 

microAOD campaign and metaconditions. Campaign changes and metaconditions changes will affect subsequent steps too. different metaconditions will make different versions of microAOD productions on the same dataset. 

CMSSW_10_6_29, flashgg dev II, MonoHiggs and then scram. 





# Setting up flashgg dev_legacy_runII with MonoHiggsToGG
1. Get flashgg (for more details [link](https://github.com/cms-analysis/flashgg/tree/dev_legacy_runII) )
   ```
   export SCRAM_ARCH=slc7_amd64_gcc700
   cmsrel CMSSW_10_6_29
   cd CMSSW_10_6_29/src
   cmsenv
   git cms-init
   cd $CMSSW_BASE/src 
   git clone -b dev_legacy_runII https://github.com/cms-analysis/flashgg 
   source flashgg/setup_flashgg.sh
   ```
   
2. Get MonoHiggsToGG (after following above steps)
   ```
   cd $CMSSW_BASE/src
   cp -r /afs/cern.ch/work/s/shdutta/public/Analysis/MHgg/2018Analysis/CMSSW_10_6_8/src/MonoHiggsToGG ./
   ```
   
3. Before building, change the location of the header file `LeptonSelection.h` to `MonoHiggsToGG/analysis/addfiles/LeptonSelection.h` in the analyzer plugin `MonoHiggsToGG/analysis/plugins/DiPhoAnalyzer_Legacy18.cc`

4. Now build
   `scram b -j 10`
   

## Brief Summary of Issue as of 22/03/2023

The MonoHiggsToGG analyzer `DiPhoAnalyzer_Legacy18.cc` has a method `flashgg::Muon::selectLooseMuons()` which has no definition in the `flashgg/Taggers/interface/LeptonSelection.h` header. The MonoHiggsToGG setup has a header file `LeptonSelection.h` in `MonoHiggsToGG/analysis/addfiles`. When this file was copied in the directory `flashgg/Taggers/interface` (with the tag `_MHgg`), a new error was encountered while compilation. The error was that the MonoHiggsToGG version of the header file was expecting the presence of the member `reco::HitPattern::numberOfHits()`. This class `reco::HitPattern` (`reco` being the namespace) is defined in the header `cmssw/DataFormats/TrackReco/interface/HitPattern.h`. Upon checking in the `cmssw` repo in github, it was found that the member `numberOfHits()` exists in `CMSSW_9_3_X` branch, but it is replaced by `numberOfAllHits()` since `CMSSW_9_4_X` branch. However, the strange thing is that MonoHiggsToGG is compatible with `CMSSW_10_6_8`.

The definition of both the methods `numberOfHits()` and `numberOfAllHits()` is exactly the same. So, just to check, changing the name of the method in the MonoHiggsToGG version of the header file and then compiling. The `numberOfHits()` error was gone but this did not compile due to multiple definitions of methods in `LeptonSelection.h` and `LeptonSelection_MHgg.h`. Tried "switching off" `LeptonSelection.h` and only keeping `LeptonSelection_MHgg.h`. But, four plugins use the `flashgg/Taggers/interface/LeptonSelection.h` which are showing error. The plugins:
```
flashgg/Taggers/plugins/VHLeptonicLoose.cc
flashgg/Taggers/plugins/VHLooseTagProducer.cc
flashgg/Taggers/plugins/DoubleHTagProducer.cc
flashgg/Taggers/plugins/VHLeptonicLoose.cc
```
Introduced new `namespace` as `MHgg` in the MonoHiggsToGG version of the header file. This removed the compilation error but the `FileInPath` error persists. After putting flags in the `DiPhoAnalyzer_Legacy18.cc`, it was seen that not even the method `beginJob()` is being executed. The error is thrown even before that. 

If the analyzer is unable to access the input root file then the following error is being thrown which is different from what I am getting in the above case. This is checked by executing `cmsRun diPhoAna_2018.py` on the working analyzer (CMSSW_10_6_8) and deliberately giving a wrong path to the input root file. 
```
Now trying to get input file
cms.untracked.vstring('/ste/user/spigazzi/flashgg/Era2018_RR-17Sep2018_v2/legacyRun2FullV2/TGJets_TuneCP5_13TeV_amcatnlo_madspin_pythia8/Era2018_RR-17Sep2018_v2-legacyRun2FullV2-v0-RunIIAutumn18MiniAOD-102X_upgrade2018_realistic_v15-v2/190717_094233/0000/myMicroAODOutputFile_1.root')
flashggDiPhotonsVtx0
----- Begin Fatal Exception 23-Mar-2023 08:12:13 CET-----------------------
An exception of category 'LogicalFileNameNotFound' occurred while
   [0] Constructing the EventProcessor
   [1] Constructing input source of type PoolSource
Exception Message:
RootFileSequenceBase::initTheFile()
Logical file name '/ste/user/spigazzi/flashgg/Era2018_RR-17Sep2018_v2/legacyRun2FullV2/TGJets_TuneCP5_13TeV_amcatnlo_madspin_pythia8/Era2018_RR-17Sep2018_v2-legacyRun2FullV2-v0-RunIIAutumn18MiniAOD-102X_upgrade2018_realistic_v15-v2/190717_094233/0000/myMicroAODOutputFile_1.root' was not found in the file catalog.
If you wanted a local file, you forgot the 'file:' prefix
before the file name in your configuration file.
----- End Fatal Exception -------------------------------------------------
```

Again, by deliberately putting erroneous path to the configuration files (variables `photonFileName` and `jetFileName`), the following errors are thrown where, the erroneous path is explicitly included in the error statements. 
```
flashggDiPhotonsVtx0
----- Begin Fatal Exception 23-Mar-2023 08:18:54 CET-----------------------
An exception of category 'ConfigFileReadError' occurred while
   [0] Processing the python configuration file named diPhoAna_2018.py
Exception Message:
 unknown python problem occurred.
RuntimeError: An exception of category 'FileInPathError' occurred.
Exception Message:
edm::FileInPath unable to find file flashgg/Tggers/data/L1prefiring_photonpt_2017BtoF.root anywhere in the search path.
The search path is defined by: CMSSW_SEARCH_PATH
${CMSSW_SEARCH_PATH} is: /afs/cern.ch/work/s/shdutta/public/Analysis/MHgg/2018Analysis/CMSSW_10_6_8/poison:/afs/cern.ch/work/s/shdutta/public/Analysis/MHgg/2018Analysis/CMSSW_10_6_8/src:/afs/cern.ch/work/s/shdutta/public/Analysis/MHgg/2018Analysis/CMSSW_10_6_8/external/slc7_amd64_gcc700/data:/cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/src:/cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/external/slc7_amd64_gcc700/data
Current directory is: /afs/cern.ch/work/s/shdutta/public/Analysis/MHgg/2018Analysis/CMSSW_10_6_8/src/MonoHiggsToGG/analysis/test/MC_sig/ggTomonoH_aa_sinp0p35_tanb1p0_MXd10_MH3_300_MH4_200


At:
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Types.py(639): insertInto
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Mixins.py(356): insertContentsInto
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Mixins.py(482): insertInto
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Modules.py(162): insertInto
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Config.py(900): _insertManyInto
  /cvmfs/cms.cern.ch/slc7_amd64_gcc700/cms/cmssw/CMSSW_10_6_8/python/FWCore/ParameterSet/Config.py(1102): fillProcessDesc
  <string>(2): <module>

----- End Fatal Exception -------------------------------------------------
```

So, most likely the paths to these config files are not the source of the problem. 

The same error as of CMSSW_10_6_29 could be reproduced in CMSSW_10_6_9 by leaving `photonFileName` completely empty. So, why is this error being encountered in the former even though the string is filled needs to be checked now. 
