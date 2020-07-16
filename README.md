# docker-jupyterlab

The intent is to generate a docker image with Jupyter Notebook, and PySpark without conda.

This project is base on https://github.com/jupyter/docker-stacks/,

## Images

- base-notebook:
    - Docker image with Spark 3.0
- pyspark-lab:
    * base on the https://github.com/jupyter/docker-stacks/ for Jupyter lab with Spark 2.4.5.

## Quick Start

### For Notebook
```bash
docker-compose build book_image
docker-compose up book
```

### For Jupyter Lab
```bash
docker-compose build lab_image
docker-compose up lab
```