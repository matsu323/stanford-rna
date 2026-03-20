# datasetのダウンロード
mkdir -p ./kaggle/input/datasets

while read ds; do
  owner="${ds%%/*}"
  slug="${ds#*/}"
  out="./kaggle/input/datasets/$owner/$slug"
  mkdir -p "$out"
  echo "Downloading $ds -> $out"
  uv run kaggle datasets download -d "$ds" -p "$out" --unzip
done <<'EOF'
kami1976/biopython-cp312
amirrezaaleyasin/biotite
qiweiyin/protenix-v1-adjusted
amirrezaaleyasin/rdkit-2025-9-5
metric/usalign
EOF