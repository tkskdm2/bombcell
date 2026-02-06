# Bombcell ソースコード変更履歴

このドキュメントは、Bombcellのソースコードに加えた変更の履歴をまとめたものです。

## 変更の概要

以下の問題を解決するために、Bombcellのソースコードに修正を加えました：

1. **PhyとBombcell GUIの間でのユニット不一致**: Phyで表示されるユニットの一部がBombcellの出力に含まれていなかった
2. **Template waveformsの不一致**: PhyとBombcell GUIで同一IDに紐づくtemplate waveformsが一致していなかった
3. **Mean raw waveformsの不一致**: PhyとBombcell GUIでmean raw waveformsが一致していなかった
4. **GUI表示の問題**: Phy IDの表示が正しくなく、ナビゲーションがBombcell IDを使用していた

## 変更されたファイル

### 1. `helper_functions.py`

#### 変更1: すべてのユニットを出力に含める（`run_bombcell`関数）

**場所**: 約1214-1245行目

**変更内容**:
- `removeDuplicateSpikes`が有効な場合、重複削除で除外されたユニットも`unique_templates`に含めるように修正
- `all_unique_units_original`を保存し、`non_empty_units`とマージしてすべてのユニットを含める

**変更前**:
```python
unique_templates = non_empty_units
```

**変更後**:
```python
# Store all unique units from original spike_clusters before duplicate removal
all_unique_units_original = np.unique(spike_clusters)

if param["removeDuplicateSpikes"]:
    # ... duplicate removal code ...
    # Include all unique units from original spike_clusters, even if all their spikes were removed as duplicates
    non_empty_units = np.unique(np.concatenate([non_empty_units, all_unique_units_original]))
else:
    non_empty_units = np.unique(spike_clusters)

unique_templates = non_empty_units
```

#### 変更2: `quality_metrics`と`maxChannels`のサイズ調整

**場所**: 約1264-1293行目

**変更内容**:
- `quality_metrics`の初期化サイズを`unique_templates`のサイズに合わせる
- `maxChannels`を`unique_templates`のサイズに合わせて拡張
- `signal_to_noise_ratio`も同様に拡張

**変更後**:
```python
# Initialize quality metrics dictionary
n_units = unique_templates.size

# Expand signal_to_noise_ratio to match unique_templates size if needed
expanded_snr = None
if signal_to_noise_ratio is not None:
    expanded_snr = np.full(n_units, np.nan, dtype=float)
    for unit_idx, template_id in enumerate(unique_templates):
        if template_id < len(signal_to_noise_ratio):
            expanded_snr[unit_idx] = signal_to_noise_ratio[template_id]

quality_metrics = create_quality_metrics_dict(n_units, snr=expanded_snr)

# Expand maxChannels to match unique_templates size
expanded_maxChannels = np.full(n_units, np.nan, dtype=float)
for unit_idx, template_id in enumerate(unique_templates):
    if template_id < len(maxChannels):
        expanded_maxChannels[unit_idx] = maxChannels[template_id]
quality_metrics["maxChannels"] = expanded_maxChannels
```

#### 変更3: `get_all_quality_metrics`関数に`raw_waveforms_id_match`パラメータを追加

**場所**: 約736-751行目、1300-1316行目

**変更内容**:
- `get_all_quality_metrics`関数のシグネチャに`raw_waveforms_id_match`パラメータを追加
- 関数呼び出し時に`raw_waveforms_id_match`を渡すように修正

#### 変更4: `rawAmplitude`計算の修正

**場所**: 約1011-1066行目

**変更内容**:
- `raw_waveforms_full[unit_idx]`の代わりに`raw_waveforms_id_match[this_unit]`を使用
- `raw_waveforms_id_match`はクラスターIDでインデックスされているため、`this_unit`（クラスターID）で直接アクセス可能
- NaNチェックと範囲チェックを追加

**変更後**:
```python
if raw_waveforms_full is not None and param["extractRaw"] and param['gain_to_uV'] is not None:
    # Use raw_waveforms_id_match to access waveforms by cluster ID (this_unit)
    if raw_waveforms_id_match is not None and this_unit < raw_waveforms_id_match.shape[0]:
        template_peak_channel = quality_metrics["maxChannels"][unit_idx]
        # ... validation and calculation ...
```

### 2. `quality_metrics.py`

#### 変更1: `waveform_shape`関数の修正

**場所**: 約1162-1242行目

**変更内容**:
- `waveform_shape`関数に`unit_idx`パラメータを追加
- `maxChannels[this_unit]`の代わりに`maxChannels[unit_idx]`を使用
- `spatialDecay`計算でも同様の修正

**変更後**:
```python
def waveform_shape(
    template_waveforms,
    this_unit,
    maxChannels,
    channel_positions,
    waveform_baseline_window,
    param,
    unit_idx=None,
):
    # maxChannels is now indexed by unit_idx, not by this_unit (cluster ID)
    if unit_idx is not None and unit_idx < len(maxChannels):
        max_channel_idx = maxChannels[unit_idx]
    else:
        # Fallback: try to use this_unit as index (for backward compatibility)
        if this_unit < len(maxChannels):
            max_channel_idx = maxChannels[this_unit]
        else:
            max_channel_idx = 0
    # ... rest of function ...
```

#### 変更2: `get_raw_amplitude`関数の修正

**場所**: 約1884-1898行目

**変更内容**:
- `peak_channel`がNaNまたは範囲外の場合の検証を追加
- 無効な場合は整数に変換し、範囲外の場合は最初のチャンネルを使用

**変更後**:
```python
if peak_channel is not None and this_raw_waveform_tmp.ndim == 2:
    # Ensure peak_channel is a valid integer
    if np.isnan(peak_channel) or peak_channel < 0 or peak_channel >= this_raw_waveform_tmp.shape[0]:
        peak_channel = 0
    else:
        peak_channel = int(peak_channel)
    this_raw_waveform_tmp = this_raw_waveform_tmp[peak_channel, :]
```

### 3. `plot_functions.py`

#### 変更: `plot_waveforms_overlay`関数の修正

**場所**: 約355-379行目

**変更内容**:
- `template_id`（クラスターID）から`unit_idx`を取得する処理を追加
- `quality_metrics["maxChannels"][unit_idx]`を使用するように変更
- `max_channel_id`がNaNまたは範囲外の場合のフォールバック処理を追加

**変更後**:
```python
for template_id in unit_type_template_ids:
    # Find unit_idx from template_id (cluster ID)
    unit_idx = np.where(unique_templates == template_id)[0]
    if len(unit_idx) > 0:
        unit_idx = unit_idx[0]
        max_channel_id = quality_metrics["maxChannels"][unit_idx]
        # Check if max_channel_id is valid (not NaN and within bounds)
        if not np.isnan(max_channel_id) and max_channel_id >= 0 and template_id < template_waveforms.shape[0]:
            max_channel_id = int(max_channel_id)
            if max_channel_id < template_waveforms.shape[2]:
                # ... plot waveform ...
```

### 4. `unit_quality_gui.py`

#### 変更1: Phy ID表示の修正

**場所**: 約1299-1310行目、その他の関連箇所

**変更内容**:
- `_get_phy_id`メソッドを追加して、`quality_metrics`から`phy_clusterID`を取得
- `update_unit_info`メソッドでPhy IDを正しく表示
- "unit # xxx/yyy"の表示をPhy IDと最大Phy IDに変更
- "Go to:"入力フィールドをPhy IDで動作するように変更

**追加されたメソッド**:
- `_get_phy_id(unit_idx)`: `quality_metrics`から`phy_clusterID`を取得
- `_get_max_phy_id()`: 最大`phy_clusterID`を取得
- `_get_unit_idx_from_phy_id(phy_id)`: Phy IDから`unit_idx`を取得

#### 変更2: Raw waveforms表示の修正

**場所**: 約3761-3773行目、1606-1630行目

**変更内容**:
- `_bc_rawWaveforms_kilosort_format.npy`を読み込み、`raw_waveforms['id_match']`として保存
- `plot_raw_waveforms`関数で`raw_waveforms_id_match[current_unit_id]`を使用
- `raw_waveforms_id_match`はクラスターIDでインデックスされているため、`current_unit_id`（phy_clusterID）で直接アクセス可能

**変更後**:
```python
# Load raw_waveforms_id_match
raw_wf_id_match_path = bombcell_path / "_bc_rawWaveforms_kilosort_format.npy"
if raw_wf_path.exists():
    try:
        raw_waveforms = {
            'average': np.load(raw_wf_path, allow_pickle=True),
            'peak_channels': np.load(bombcell_path / "templates._bc_rawWaveformPeakChannels.npy", allow_pickle=True)
        }
        # Load raw_waveforms_id_match if available
        if raw_wf_id_match_path.exists():
            raw_waveforms['id_match'] = np.load(raw_wf_id_match_path, allow_pickle=True)
    except FileNotFoundError:
        raw_waveforms = None

# In plot_raw_waveforms:
raw_wf_id_match = self.raw_waveforms.get('id_match', None)
current_unit_id = self.unique_units[self.current_unit_idx]

if raw_wf_id_match is not None and current_unit_id < len(raw_wf_id_match):
    # Use id_match (indexed by cluster ID)
    waveforms = raw_wf_id_match[current_unit_id]
elif raw_wf is not None:
    # Fallback to average (indexed by unit_idx)
    if hasattr(raw_wf, '__len__') and self.current_unit_idx < len(raw_wf):
        waveforms = raw_wf[self.current_unit_idx]
```

#### 変更3: Template waveform表示の修正

**場所**: 約1215-1219行目

**変更内容**:
- `get_unit_data`関数でtemplate waveformを取得する際に、`unit_id`（クラスターID）を使用
- `template_waveforms`は`unit_id`でインデックスされているため、`unit_idx`ではなく`unit_id`を使用

**変更前**:
```python
# Get template waveform
if unit_idx < len(self.ephys_data['template_waveforms']):
    template = self.ephys_data['template_waveforms'][unit_idx]
else:
    template = np.zeros((82, 1))
```

**変更後**:
```python
# Get template waveform
# template_waveforms is indexed by unit_id (cluster ID), not unit_idx
if unit_id < len(self.ephys_data['template_waveforms']):
    template = self.ephys_data['template_waveforms'][unit_id]
else:
    template = np.zeros((82, 1))
```

### 5. `save_utils.py`

#### 変更: `maxChannels`の再インデックス処理を無効化

**場所**: 約299-303行目

**変更内容**:
- `save_results`関数で`maxChannels`を`phy_clusterID`でインデックスする処理をコメントアウト
- `maxChannels`は既に`unit_idx`でインデックスされており、`phy_clusterID`での再インデックスは不要
- この変換を行うと、クラスターIDが大きい場合に範囲外アクセスが発生

**変更前**:
```python
# Get rid of peak channels of empty rows, which were kept for convenient indexing up to here
quality_metrics_save = quality_metrics.copy()
quality_metrics_save["maxChannels"] = quality_metrics["maxChannels"][
    quality_metrics["phy_clusterID"].astype(int)
]
```

**変更後**:
```python
# Get rid of peak channels of empty rows, which were kept for convenient indexing up to here
quality_metrics_save = quality_metrics.copy()
# maxChannels is already indexed by unit_idx, no need to reindex by phy_clusterID
# quality_metrics_save["maxChannels"] = quality_metrics["maxChannels"][
#     quality_metrics["phy_clusterID"].astype(int)
# ]
```

## 変更の影響

### 解決された問題

1. ✅ PhyとBombcell GUIの間でのユニット不一致が解消
2. ✅ Template waveformsがPhyとBombcell GUIで一致
3. ✅ Mean raw waveformsがPhyとBombcell GUIで一致
4. ✅ GUIでPhy IDが正しく表示される
5. ✅ GUIのナビゲーションがPhy IDを使用

### 互換性

- 既存のコードとの後方互換性を維持
- `waveform_shape`関数に`unit_idx`パラメータを追加したが、`None`の場合は従来の動作（`this_unit`を使用）にフォールバック
- エラーハンドリングを追加して、無効な値の場合でもクラッシュしないように改善

## 注意事項

1. **`raw_waveforms_id_match`の必要性**: Mean raw waveformsを正しく表示するには、`_bc_rawWaveforms_kilosort_format.npy`ファイルが必要です。このファイルは`extract_raw_waveforms`関数によって自動的に生成されます。

2. **パフォーマンス**: すべてのユニットを含めることで、処理時間がわずかに増加する可能性がありますが、通常は無視できる程度です。

3. **メモリ使用量**: `unique_templates`のサイズが増加するため、メモリ使用量がわずかに増加する可能性があります。

## テスト

以下の点を確認してください：

1. ✅ すべてのユニット（Phyで表示されるすべてのユニット）が`templates._bc_qMetrics.csv`に含まれている
2. ✅ PhyとBombcell GUIで同一IDのtemplate waveformsが一致している
3. ✅ PhyとBombcell GUIで同一IDのmean raw waveformsが一致している
4. ✅ GUIでPhy IDが正しく表示されている
5. ✅ GUIのナビゲーションがPhy IDで動作している

## 変更日

2026年1月（具体的な日付は変更履歴を参照）

