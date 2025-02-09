# ChEMBL Multitask Nerual Network model

- Based on the blogpost: http://chembl.blogspot.com/2019/05/multi-task-neural-network-on-chembl.html
- Model available in KNIME thanks to Greg Landrum: https://www.knime.com/blog/interactive-bioactivity-prediction-with-multitask-neural-networks

The model is exported to the ONNX format so it can be used in any programming language able to generate fingerprints with RDKit

# Example to predict in Python using the ONNX Runtime

```Python
import onnxruntime
import numpy as np
from rdkit import Chem
from rdkit.Chem import rdMolDescriptors

FP_SIZE = 1024
RADIUS = 2

def calc_morgan_fp(smiles):
    mol = Chem.MolFromSmiles(smiles)
    fp = rdMolDescriptors.GetMorganFingerprintAsBitVect(
        mol, RADIUS, nBits=FP_SIZE)
    a = np.zeros((0,), dtype=np.float32)
    Chem.DataStructs.ConvertToNumpyArray(fp, a)
    return a

def format_preds(preds, targets):
    preds = np.concatenate(preds).ravel()
    np_preds = [(tar, pre) for tar, pre in zip(targets, preds)]
    dt = [('chembl_id','|U20'), ('pred', '<f4')]
    np_preds = np.array(np_preds, dtype=dt)
    np_preds[::-1].sort(order='pred')
    return np_preds

# load the model
ort_session = onnxruntime.InferenceSession("trained_models/chembl_33_model/chembl_33_multitask.onnx", providers=['CPUExecutionProvider'])

# calculate the FPs
smiles = 'CN(C)CCc1c[nH]c2ccc(C[C@H]3COC(=O)N3)cc12'
descs = calc_morgan_fp(smiles)

# run the prediction
ort_inputs = {ort_session.get_inputs()[0].name: descs}
preds = ort_session.run(None, ort_inputs)

# example of how the output of the model can be formatted
preds = format_preds(preds, [o.name for o in ort_session.get_outputs()])
```

# In Julia using [RDKitMinimalLib.jl](https://github.com/eloyfelix/RDKitMinimalLib.jl) and [ONNX.jl](https://github.com/FluxML/ONNX.jl)

```julia
import RDKitMinimalLib: get_mol, get_morgan_fp
import Umlaut: play!
import ONNX
import JSON

path = "chembl_31_multitask.onnx"
targets = JSON.parsefile("targets_31.json")

# dummy input
dummy = rand(Float32, 1024, 1)
# load the model
mt_chembl = ONNX.load(path, dummy)

# load molecule and calc morgan fingerprint
mol = get_mol("CC(=O)Oc1ccccc1C(=O)O")
fp_details = Dict{String, Any}("nBits" => 1024, "radius" => 2)
mfp = get_morgan_fp(mol, fp_details)

# convert the bitstring to a 1024×1 Matrix{Float32}
mfp = map(x->parse(Float32,string(x)),collect(mfp))
mfp = reshape(mfp, (length(mfp), 1))

# test a molecule
pred = play!(mt_chembl, mfp)
pred = collect(Iterators.flatten(pred))

res = tuple.(targets, pred)
res = sort(res, by=res->res[2], rev=true)
```

# C++ REST microservice

https://github.com/eloyfelix/pistache_predictor

# Try it online!

Using both RDKit Javascript MinimalLib and ONNX.js. Hosted in github pages: https://chembl.github.io/chembl_multitask_model
