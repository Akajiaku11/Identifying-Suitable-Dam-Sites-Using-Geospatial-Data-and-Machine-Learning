import rasterio
import geopandas as gpd
import numpy as np
import pandas as pd
from rasterio.plot import show
from sklearn.svm import SVC
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, confusion_matrix
from scipy.stats import zscore

# Define paths for datasets
DEM_PATH = 'data/srtm_dem.tif'
SLOPE_PATH = 'data/slope.tif'
STREAM_ORDER_PATH = 'data/stream_order.tif'
LANDCOVER_PATH = 'data/landcover.tif'
RAINFALL_PATH = 'data/rainfall.tif'
SOIL_PATH = 'data/soil_data.tif'
GEOLOGY_PATH = 'data/geology_data.tif'

# Load raster data
def load_raster(file_path):
    with rasterio.open(file_path) as src:
        return src.read(1), src.meta

# Preprocess raster data
def preprocess_raster(data):
    """ Normalize data and handle no data values """
    data = np.where(data == -9999, np.nan, data)  # Replace no data value
    return np.nan_to_num((data - np.nanmin(data)) / (np.nanmax(data) - np.nanmin(data)))

# Load and preprocess all criteria
dem, dem_meta = load_raster(DEM_PATH)
dem = preprocess_raster(dem)

slope, _ = load_raster(SLOPE_PATH)
slope = preprocess_raster(slope)

stream_order, _ = load_raster(STREAM_ORDER_PATH)
stream_order = preprocess_raster(stream_order)

landcover, _ = load_raster(LANDCOVER_PATH)
landcover = preprocess_raster(landcover)

rainfall, _ = load_raster(RAINFALL_PATH)
rainfall = preprocess_raster(rainfall)

soil, _ = load_raster(SOIL_PATH)
soil = preprocess_raster(soil)

geology, _ = load_raster(GEOLOGY_PATH)
geology = preprocess_raster(geology)

# Combine criteria layers into a DataFrame
criteria = {
    'elevation': dem.flatten(),
    'slope': slope.flatten(),
    'stream_order': stream_order.flatten(),
    'landcover': landcover.flatten(),
    'rainfall': rainfall.flatten(),
    'soil': soil.flatten(),
    'geology': geology.flatten(),
}

criteria_df = pd.DataFrame(criteria)
criteria_df.dropna(inplace=True)

# Assign AHP weights (example weights from your study)
weights = {
    'elevation': 0.13,
    'slope': 0.21,
    'stream_order': 0.34,
    'landcover': 0.08,
    'rainfall': 0.14,
    'soil': 0.05,
    'geology': 0.05
}

# Weighted Sum Model for suitability
criteria_df['suitability_score'] = (
    criteria_df['elevation'] * weights['elevation'] +
    criteria_df['slope'] * weights['slope'] +
    criteria_df['stream_order'] * weights['stream_order'] +
    criteria_df['landcover'] * weights['landcover'] +
    criteria_df['rainfall'] * weights['rainfall'] +
    criteria_df['soil'] * weights['soil'] +
    criteria_df['geology'] * weights['geology']
)

# Normalize suitability score
criteria_df['suitability_score'] = zscore(criteria_df['suitability_score'])

# Prepare data for SVM classification
X = criteria_df.drop(columns=['suitability_score'])
y = np.where(criteria_df['suitability_score'] > 0.5, 1, 0)  # Threshold suitability

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Apply SVM for classification
svm_model = SVC(kernel='rbf', C=1, gamma='auto')
svm_model.fit(X_train, y_train)
y_pred = svm_model.predict(X_test)

# Validate the model
accuracy = accuracy_score(y_test, y_pred)
conf_matrix = confusion_matrix(y_test, y_pred)
print("Model Accuracy:", accuracy)
print("Confusion Matrix:\n", conf_matrix)

# Save suitability map back to raster format
def save_raster(output_path, data, meta):
    meta.update(dtype='float32', count=1)
    with rasterio.open(output_path, 'w', **meta) as dst:
        dst.write(data, 1)

# Reshape suitability scores to original raster shape
suitability_map = criteria_df['suitability_score'].values.reshape(dem.shape)
save_raster('output/suitability_map.tif', suitability_map, dem_meta)
print("Suitability map saved successfully.")
