# FloatChat - AI-Powered Ocean Data Assistant

FloatChat is an intelligent conversational system that makes oceanographic data from the ARGO float network accessible through natural language. Ask questions in plain English and get instant insights about ocean temperature, salinity, biogeochemical parameters, and more.

## 🌊 What is FloatChat?

FloatChat transforms complex ocean data into conversational insights. Instead of writing SQL queries or processing NetCDF files manually, simply ask:
- "What's the temperature in the Arabian Sea?"
- "Show me oxygen levels near Mumbai"
- "Are there any marine heatwaves active right now?"

The system uses AI to understand your questions, retrieve relevant data from ARGO floats, and provide clear, contextual answers.

## ✨ Key Features

- **Natural Language Interface**: Ask questions in plain English
- **Dual Database Architecture**: PostgreSQL for structured queries + Pinecone for semantic search
- **BGC Parameter Support**: Temperature, salinity, oxygen, chlorophyll, pH, nitrate, and more
- **Geospatial Queries**: Search by region, coordinates, or radius
- **Quality Control**: Automatic filtering of bad data using ARGO QC flags
- **Forecasting** (Coming Soon): 7-day and 30-day predictions for ocean parameters
- **Marine Heatwave Detection** (Coming Soon): Automated detection using validated algorithms
- **Interactive Visualization** (Coming Soon): Web-based maps and charts

## 📋 Prerequisites

Before setting up FloatChat, ensure you have:

- **Python 3.8+** installed
- **PostgreSQL** database (local or cloud)
- **Pinecone account** (free tier available at https://www.pinecone.io/)
- **Git** for cloning the repository
- **pip** or **conda** for package management

## 🚀 Quick Start Guide

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd Floatchat-main
```

### Step 2: Set Up Python Environment

Create and activate a virtual environment:

```bash
# Using venv
python -m venv .venv

# Activate on Windows
.venv\Scripts\activate

# Activate on Linux/Mac
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### Step 3: Configure Environment Variables

Create a `.env` file in the `Data preprocessing and testing files` directory:

```bash
cd "Data preprocessing and testing files"
```

Create `.env` file with the following content:

```env
# PostgreSQL Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=argo_floats
DB_USER=your_username
DB_PASSWORD=your_password

# Pinecone Configuration
PINECONE_API_KEY=your_pinecone_api_key
PINECONE_ENVIRONMENT=your_pinecone_environment
PINECONE_INDEX_NAME=argo-floats

# Optional: LLM Configuration
OPENAI_API_KEY=your_openai_key  # If using OpenAI
# OR
ANTHROPIC_API_KEY=your_anthropic_key  # If using Claude
```

### Step 4: Set Up PostgreSQL Database

Create the database and tables:

```sql
-- Connect to PostgreSQL
psql -U your_username

-- Create database
CREATE DATABASE argo_floats;

-- Connect to the database
\c argo_floats

-- Run the table creation script
\i "Data preprocessing and testing files/create_postgresesql_tables.sql"
```

### Step 5: Configure File Paths

**IMPORTANT**: Update file paths in Python scripts to match your folder structure.

The following files contain hardcoded paths that need to be updated:

1. **BCGparameters.py** - Processes BGC float data
2. **NON_BCGparameters.py** - Processes standard float data
3. **generate_summaries.py** - Generates data summaries
4. **pinecone_upload.py** - Uploads to Pinecone
5. **postgres_raw_upload.py** - Uploads to PostgreSQL
6. **process_raw_csvs_for_postgrese.py** - Processes CSV files

**Current paths** (relative to `F:\Floatchat-main`):
```python
BASE_PATH = "Data preprocessing and testing files/"
```

**If your working directory is different**, update all paths accordingly. For example, if your project is at `C:\Projects\FloatChat`:

```python
BASE_PATH = "C:/Projects/FloatChat/Data preprocessing and testing files/"
```

**Search for these path patterns** in the files:
- `Data preprocessing and testing files/Data/BCG floats/`
- `Data preprocessing and testing files/Data/Non-BCG floats/`

Replace with your actual folder structure.

## 📊 Data Preprocessing Pipeline

### Understanding the Data Structure

FloatChat processes two types of ARGO float data:

1. **BGC Floats**: Biogeochemical floats with 10+ parameters
   - Location: `Data preprocessing and testing files/Data/BCG floats/`
   - Parameters: Temperature, salinity, oxygen, chlorophyll, pH, nitrate, etc.

2. **Non-BGC Floats**: Standard floats with core parameters
   - Location: `Data preprocessing and testing files/Data/Non-BCG floats/`
   - Parameters: Temperature, salinity, pressure

### Data Processing Steps

#### 1. Convert NetCDF to CSV

**For BGC floats**:
```bash
cd "Data preprocessing and testing files"
python BCGparameters.py
```

This script:
- Reads NetCDF files from `Data/BCG floats/netcdf files/`
- Converts to CSV format
- Outputs to `Data/BCG floats/final csv files/`

**For Non-BGC floats**:
```bash
python NON_BCGparameters.py
```

This script:
- Reads NetCDF files from `Data/Non-BCG floats/netcdf files/`
- Converts to CSV format
- Outputs to `Data/Non-BCG floats/final csv files/`

#### 2. Upload to PostgreSQL

```bash
python postgres_raw_upload.py
```

This script:
- Reads CSV files from both BGC and Non-BGC directories
- Uploads data to PostgreSQL in batches (30,000 records per batch)
- Handles errors gracefully and logs progress

**Expected output**:
```
Processing BGC floats...
Uploaded 45,230 records from float 1902367
Uploaded 38,910 records from float 1902372
...
Processing Non-BGC floats...
Uploaded 52,100 records from float 2901234
...
Total records uploaded: 1,234,567
```

#### 3. Generate Embeddings and Upload to Pinecone

```bash
python pinecone_upload.py
```

This script:
- Generates 384-dimensional embeddings using sentence-transformers
- Uploads vectors to Pinecone with metadata
- Processes in batches (100 vectors per batch)

**Expected output**:
```
Generating embeddings for BGC floats...
Uploaded batch 1/150 to Pinecone
Uploaded batch 2/150 to Pinecone
...
Total vectors uploaded: 15,000
```

## 🎯 Running FloatChat

### Start the MCP Server

```bash
cd "Data preprocessing and testing files"
python argo_mcp.py
```

The MCP server provides tools for:
- Float discovery and metadata retrieval
- Parameter data queries (temperature, salinity, oxygen, etc.)
- Geospatial searches
- Statistical summaries
- Semantic search

### Using the Chat Interface

Once the MCP server is running, you can interact with FloatChat through your preferred LLM interface (Claude, ChatGPT, etc.) configured to use the MCP server.

**Example queries**:
```
"List all BGC floats in the Arabian Sea"
"Get temperature data for float 1902367 from January to June 2023"
"Show me oxygen levels within 100km of coordinates 15.5N, 68.2E"
"What's the average salinity in the Bay of Bengal?"
"Find floats with chlorophyll measurements near Mumbai"
```

## 📁 Project Structure

```
Floatchat-main/
├── Data preprocessing and testing files/
│   ├── Data/
│   │   ├── BCG floats/
│   │   │   ├── netcdf files/          # Raw NetCDF files
│   │   │   ├── raw csv/               # Initial CSV conversion
│   │   │   ├── final csv files/       # Processed CSV for upload
│   │   │   └── raw_supabase_upload_csvs/
│   │   └── Non-BCG floats/
│   │       ├── netcdf files/
│   │       ├── raw csv/
│   │       └── final csv files/
│   ├── BCGparameters.py               # BGC data processor
│   ├── NON_BCGparameters.py           # Non-BGC data processor
│   ├── postgres_raw_upload.py         # PostgreSQL uploader
│   ├── pinecone_upload.py             # Pinecone uploader
│   ├── argo_mcp.py                    # MCP server
│   ├── generate_summaries.py          # Data summarization
│   ├── process_raw_csvs_for_postgrese.py
│   ├── create_postgresesql_tables.sql # Database schema
│   └── .env                           # Environment variables
├── requirements.md                     # Project requirements
├── design.md                          # Technical design document
└── README.md                          # This file
```

## 🔧 Configuration Guide

### Adjusting for Your Folder Structure

If your project is located in a different directory or you want to organize data differently:

1. **Update base paths** in all Python files:
   ```python
   # Find and replace in:
   # - BCGparameters.py
   # - NON_BCGparameters.py
   # - postgres_raw_upload.py
   # - pinecone_upload.py
   # - generate_summaries.py
   
   # Old path:
   BASE_PATH = "Data preprocessing and testing files/"
   
   # New path (example):
   BASE_PATH = "/your/custom/path/Data preprocessing and testing files/"
   ```

2. **Update data directories** if you reorganize the Data folder:
   ```python
   # BGC data paths
   NETCDF_DIR = "Data/BCG floats/netcdf files/"
   CSV_OUTPUT_DIR = "Data/BCG floats/final csv files/"
   
   # Non-BGG data paths
   NETCDF_DIR = "Data/Non-BCG floats/netcdf files/"
   CSV_OUTPUT_DIR = "Data/Non-BCG floats/final csv files/"
   ```

3. **Verify paths** before running scripts:
   ```python
   import os
   print(os.path.exists("Data preprocessing and testing files/Data/BCG floats/"))
   ```

### Database Configuration

**PostgreSQL connection settings** in `.env`:
```env
DB_HOST=localhost          # Change if using remote database
DB_PORT=5432              # Default PostgreSQL port
DB_NAME=argo_floats       # Database name
DB_USER=your_username     # Your PostgreSQL username
DB_PASSWORD=your_password # Your PostgreSQL password
```

**Pinecone configuration** in `.env`:
```env
PINECONE_API_KEY=your_key
PINECONE_ENVIRONMENT=us-west1-gcp  # Or your region
PINECONE_INDEX_NAME=argo-floats
```

## 🧪 Testing Your Setup

### 1. Test Database Connection

```python
import psycopg2
from dotenv import load_dotenv
import os

load_dotenv()

conn = psycopg2.connect(
    host=os.getenv('DB_HOST'),
    port=os.getenv('DB_PORT'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

print("✅ PostgreSQL connection successful!")
conn.close()
```

### 2. Test Pinecone Connection

```python
import pinecone
from dotenv import load_dotenv
import os

load_dotenv()

pinecone.init(
    api_key=os.getenv('PINECONE_API_KEY'),
    environment=os.getenv('PINECONE_ENVIRONMENT')
)

index = pinecone.Index(os.getenv('PINECONE_INDEX_NAME'))
print(f"✅ Pinecone connection successful! Index stats: {index.describe_index_stats()}")
```

### 3. Test Data Processing

```bash
# Process a single BGC float as a test
cd "Data preprocessing and testing files"
python BCGparameters.py --test --float-id 1902367
```

## 🚀 AWS Deployment (Coming Soon)

FloatChat is designed for AWS cloud deployment with:
- **Lambda functions** for serverless data processing
- **RDS PostgreSQL** for managed database
- **S3** for data lake storage
- **API Gateway** for RESTful API
- **SageMaker** for ML model hosting
- **CloudFront** for global CDN

See `design.md` for complete AWS architecture details.

## 📚 Additional Resources

- **Requirements Document**: See `requirements.md` for detailed feature specifications
- **Design Document**: See `design.md` for technical architecture
- **ARGO Float Program**: https://argo.ucsd.edu/
- **ARGO Data Access**: https://www.ifremer.fr/erddap/

## 🤝 Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## 📄 License

[Add your license information here]

## 🐛 Troubleshooting

### Common Issues

**Issue**: "Module not found" errors
```bash
# Solution: Install missing dependencies
pip install -r requirements.txt
```

**Issue**: "Permission denied" when accessing database
```bash
# Solution: Check PostgreSQL user permissions
psql -U postgres
GRANT ALL PRIVILEGES ON DATABASE argo_floats TO your_username;
```

**Issue**: "File not found" errors in Python scripts
```bash
# Solution: Verify and update file paths in the scripts
# Check that paths match your actual folder structure
```

**Issue**: Pinecone connection timeout
```bash
# Solution: Check API key and environment settings in .env
# Verify your Pinecone account is active
```

**Issue**: Out of memory during data processing
```bash
# Solution: Process data in smaller batches
# Reduce BATCH_SIZE in upload scripts
```

## 📞 Support

For questions or issues:
1. Check the troubleshooting section above
2. Review `design.md` for technical details
3. Open an issue on GitHub
4. Contact the development team

---

**Built with ❤️ for ocean data accessibility and climate research**
