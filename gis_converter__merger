# Import required libraries
import streamlit as st  # For creating web interface
import geopandas as gpd  # For handling geospatial data
import pandas as pd  # For data manipulation
import zipfile  # For handling zip files
import tempfile  # For creating temporary directories
import os  # For file and path operations
from pathlib import Path  # For path manipulations

def merge_files(files, input_formats, output_format):
    """
    Merge multiple geospatial files into a single GeoDataFrame
    Args:
        files: List of uploaded files
        input_formats: List of file formats corresponding to each file
        output_format: Desired output format
    Returns:
        GeoDataFrame containing merged data or None if error occurs
    """
    gdfs = []  # List to store individual GeoDataFrames
    for file, format in zip(files, input_formats):
        try:
            if format == "Shapefile":
                # Handle shapefile which comes as a zip containing multiple files
                with tempfile.TemporaryDirectory() as tmpdir:
                    with zipfile.ZipFile(file, "r") as zip_ref:
                        zip_ref.extractall(tmpdir)
                    shp_files = list(Path(tmpdir).glob("*.shp"))
                    if shp_files:
                        gdf = gpd.read_file(str(shp_files[0]))
                        gdfs.append(gdf)
            elif format == "KML/KMZ":
                # Read KML/KMZ file using KML driver
                gdf = gpd.read_file(file, driver="KML")
                gdfs.append(gdf)
            elif format == "GeoJSON":
                # Read GeoJSON file
                gdf = gpd.read_file(file, driver="GeoJSON")
                gdfs.append(gdf)
            else:  # CSV format
                # Read CSV and convert to GeoDataFrame if it has geometry column
                df = pd.read_csv(file)
                if "geometry" in df.columns:
                    gdf = gpd.GeoDataFrame(df, geometry="geometry")
                    gdfs.append(gdf)
                else:
                    st.error(f"CSV file {file.name} must contain a geometry column")
                    return None
        except Exception as e:
            st.error(f"Error processing {file.name}: {str(e)}")
            return None
            
    # Combine all GeoDataFrames into one
    merged_gdf = pd.concat(gdfs, ignore_index=True)
    return merged_gdf

def save_output(gdf, output_format, original_filename):
    """
    Save the merged GeoDataFrame in the specified output format
    Args:
        gdf: GeoDataFrame to save
        output_format: Desired output format
        original_filename: Base name for the output file
    """
    if output_format == "Shapefile":
        # Save as shapefile (requires creating zip with all component files)
        with tempfile.TemporaryDirectory() as tmpdir:
            output_path = os.path.join(tmpdir, f"{original_filename}.shp")
            gdf.to_file(output_path)
            zip_path = os.path.join(tmpdir, f"{original_filename}.zip")
            with zipfile.ZipFile(zip_path, "w") as zipf:
                for file in Path(tmpdir).glob(f"{original_filename}.*"):
                    zipf.write(file, file.name)
            with open(zip_path, "rb") as f:
                st.download_button(
                    "Download Merged Shapefile (ZIP)",
                    f,
                    file_name=f"{original_filename}.zip",
                )
    elif output_format == "KML":
        # Save as KML file
        gdf.to_file(f"{original_filename}.kml", driver="KML")
        with open(f"{original_filename}.kml", "rb") as f:
            st.download_button(
                "Download Merged KML", 
                f, 
                file_name=f"{original_filename}.kml"
            )
    elif output_format == "GeoJSON":
        # Save as GeoJSON file
        st.download_button(
            "Download Merged GeoJSON",
            gdf.to_json(),
            file_name=f"{original_filename}.geojson",
        )
    else:  # CSV format
        # Save as CSV file
        st.download_button(
            "Download Merged CSV",
            gdf.to_csv(index=False),
            file_name=f"{original_filename}.csv",
        )

def main():
    """Main function to run the Streamlit application"""
    st.title("Geospatial File Format Converter & Merger")
    
    # Create sidebar navigation
    page = st.sidebar.selectbox("Select Page", ["Format Converter", "Format Merger"])
    
    if page == "Format Converter":
        st.write("Convert between different geospatial file formats")

        # Input format selection dropdown
        input_format = st.selectbox(
            "Select input file format", ["Shapefile", "KML/KMZ", "GeoJSON", "CSV"]
        )

        # File uploader that adapts to selected input format
        if input_format == "Shapefile":
            uploaded_file = st.file_uploader(
                "Upload ZIP containing shapefile", type=["zip"]
            )
        elif input_format == "KML/KMZ":
            uploaded_file = st.file_uploader("Upload KML/KMZ file", type=["kml", "kmz"])
        elif input_format == "GeoJSON":
            uploaded_file = st.file_uploader(
                "Upload GeoJSON file", type=["geojson", "json"]
            )
        else:
            uploaded_file = st.file_uploader("Upload CSV file", type=["csv"])

        # Output format selection dropdown
        output_format = st.selectbox(
            "Select output file format", ["Shapefile", "KML", "GeoJSON", "CSV"]
        )

        # Process uploaded file when available
        if uploaded_file is not None:
            try:
                # Extract original filename without extension
                original_filename = Path(uploaded_file.name).stem

                # Read the input file based on its format
                if input_format == "Shapefile":
                    with tempfile.TemporaryDirectory() as tmpdir:
                        with zipfile.ZipFile(uploaded_file, "r") as zip_ref:
                            zip_ref.extractall(tmpdir)
                        shp_files = list(Path(tmpdir).glob("*.shp"))
                        if shp_files:
                            gdf = gpd.read_file(str(shp_files[0]))
                elif input_format == "KML/KMZ":
                    gdf = gpd.read_file(uploaded_file, driver="KML")
                elif input_format == "GeoJSON":
                    gdf = gpd.read_file(uploaded_file, driver="GeoJSON")
                else:
                    df = pd.read_csv(uploaded_file)
                    if "geometry" in df.columns:
                        gdf = gpd.GeoDataFrame(df, geometry="geometry")
                    else:
                        st.error("CSV must contain a geometry column")
                        return

                # Save the file in the selected output format
                save_output(gdf, output_format, original_filename)

            except Exception as e:
                st.error(f"Error processing file: {str(e)}")

    else:  # Format Merger page
        st.write("Merge multiple geospatial files")
        
        # Get number of files to merge
        file_count = st.number_input("How many files do you want to merge?", min_value=2, max_value=10, value=2)
        same_format = st.radio("Are all files in the same format?", ["Yes", "No"])
        
        files = []  # Store uploaded files
        formats = []  # Store file formats
        
        if same_format == "Yes":
            # Handle case where all input files have same format
            input_format = st.selectbox(
                "Select input file format", ["Shapefile", "KML/KMZ", "GeoJSON", "CSV"]
            )
            formats = [input_format] * file_count
            
            # Create file uploaders for each file
            for i in range(file_count):
                if input_format == "Shapefile":
                    file = st.file_uploader(
                        f"Upload ZIP containing shapefile #{i+1}", type=["zip"], key=f"same_format_{i}"
                    )
                elif input_format == "KML/KMZ":
                    file = st.file_uploader(f"Upload KML/KMZ file #{i+1}", type=["kml", "kmz"], key=f"same_format_{i}")
                elif input_format == "GeoJSON":
                    file = st.file_uploader(
                        f"Upload GeoJSON file #{i+1}", type=["geojson", "json"], key=f"same_format_{i}"
                    )
                else:
                    file = st.file_uploader(f"Upload CSV file #{i+1}", type=["csv"], key=f"same_format_{i}")
                files.append(file)
        else:
            # Handle case where input files can have different formats
            for i in range(file_count):
                format = st.selectbox(
                    f"Select format for file #{i+1}", 
                    ["Shapefile", "KML/KMZ", "GeoJSON", "CSV"],
                    key=f"format_{i}"
                )
                formats.append(format)
                
                # Create file uploader based on selected format
                if format == "Shapefile":
                    file = st.file_uploader(
                        f"Upload ZIP containing shapefile #{i+1}", type=["zip"], key=f"diff_format_{i}"
                    )
                elif format == "KML/KMZ":
                    file = st.file_uploader(f"Upload KML/KMZ file #{i+1}", type=["kml", "kmz"], key=f"diff_format_{i}")
                elif format == "GeoJSON":
                    file = st.file_uploader(
                        f"Upload GeoJSON file #{i+1}", type=["geojson", "json"], key=f"diff_format_{i}"
                    )
                else:
                    file = st.file_uploader(f"Upload CSV file #{i+1}", type=["csv"], key=f"diff_format_{i}")
                files.append(file)

        # Output format selection for merged file
        output_format = st.selectbox(
            "Select output format for merged file", 
            ["Shapefile", "KML", "GeoJSON", "CSV"]
        )

        # Process files only when all are uploaded
        if all(files):  # Only proceed if all files are uploaded
            merged_gdf = merge_files(files, formats, output_format)
            if merged_gdf is not None:
                save_output(merged_gdf, output_format, "merged_file")

if __name__ == "__main__":
    main()
