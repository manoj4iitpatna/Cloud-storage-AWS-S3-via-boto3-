Cloud Storage using AWS S3 (Python + boto3)

This project demonstrates how to upload, store, manage, and retrieve files from Amazon Web Services (AWS) S3 using Python and boto3.
It can be used for cloud-based storage systems, data pipelines, backups, automation, and API integration with services like FastAPI, Flask, or Lambda.

🔥 Features

Upload files to AWS S3 programmatically

List stored files inside a bucket

Download or retrieve files when needed

Uses Python boto3 — AWS official SDK

Easy to integrate with APIs or microservices

Scalable cloud storage for any size of data

🧠 What is AWS S3?

Amazon Simple Storage Service (AWS S3) is a secure and scalable cloud storage service used for:

Purpose	Examples
Backup & Archiving	Images, datasets, logs, media
Web/Application Storage	Static files for apps & websites
Data Engineering Pipelines	ML datasets, result outputs
Disaster Recovery	Secure encrypted storage anywhere

S3 provides 99.999999999% (11-nines) durability, unlimited scalability, encryption, global availability, and low-cost storage.

🚀 Technology Used
Tool	Purpose
Python	Programming language
boto3	SDK to interact with AWS
AWS S3	Cloud object storage
from __future__ import annotations

import os
from dataclasses import dataclass


@dataclass
class S3Config:
    """Configuration values for connecting to an S3 bucket."""
    bucket_name: str
    region: str = "ap-south-1"


def get_config() -> S3Config:
    """Load configuration from environment variables.

    Expected environment variables:
    - S3_BUCKET_NAME: Name of the S3 bucket to use.
    - AWS_REGION (optional): AWS region name, defaults to 'ap-south-1'.
    """
    bucket_name = os.getenv("S3_BUCKET_NAME")
    if not bucket_name:
        raise RuntimeError("S3_BUCKET_NAME environment variable is not set.")

    region = os.getenv("AWS_REGION", "ap-south-1")

    return S3Config(bucket_name=bucket_name, region=region)

import mimetypes
from pathlib import Path
from typing import List, Optional

import boto3
from botocore.exceptions import BotoCoreError, ClientError

from .config import get_config


def get_s3_client():
    """Create and return a low-level S3 client using default AWS credentials chain.

    boto3 will automatically read credentials from:
    - Environment variables
    - ~/.aws/credentials
    - IAM role (if running on EC2 / ECS / Lambda)
    """
    cfg = get_config()
    return boto3.client("s3", region_name=cfg.region)


def upload_file(local_path: Path, key: Optional[str] = None) -> str:
    """Upload a local file to S3.

    :param local_path: Path to the local file.
    :param key: Destination key in the S3 bucket. If None, uses 'uploads/<filename>'.
    :return: Public-style URL of the uploaded object (if bucket is public).
    """
    cfg = get_config()
    s3 = get_s3_client()

    if not local_path.is_file():
        raise FileNotFoundError(f"File not found: {local_path}")

    if key is None:
        key = f"uploads/{local_path.name}"

    content_type, _ = mimetypes.guess_type(local_path.name)
    extra_args = {}
    if content_type:
        extra_args["ContentType"] = content_type

    try:
        with local_path.open("rb") as f:
            s3.upload_fileobj(
                Fileobj=f,
                Bucket=cfg.bucket_name,
                Key=key,
                ExtraArgs=extra_args or None,
            )
    except (BotoCoreError, ClientError) as e:
        raise RuntimeError(f"Error uploading file to S3: {e}") from e

    url = f"https://{cfg.bucket_name}.s3.{cfg.region}.amazonaws.com/{key}"
    return url


def list_files(prefix: str = "") -> List[str]:
    """List object keys in the configured S3 bucket.

    :param prefix: Optional key prefix to filter results.
    :return: List of object keys.
    """
    cfg = get_config()
    s3 = get_s3_client()

    keys: List[str] = []
    continuation_token: Optional[str] = None

    try:
        while True:
            kwargs = {
                "Bucket": cfg.bucket_name,
                "Prefix": prefix,
                "MaxKeys": 1000,
            }
            if continuation_token:
                kwargs["ContinuationToken"] = continuation_token

            response = s3.list_objects_v2(**kwargs)
            contents = response.get("Contents", [])
            for obj in contents:
                keys.append(obj["Key"])

            if response.get("IsTruncated"):
                continuation_token = response.get("NextContinuationToken")
            else:
                break

    except (BotoCoreError, ClientError) as e:
        raise RuntimeError(f"Error listing S3 objects: {e}") from e

    return keys


def download_file(key: str, dest_path: Path) -> None:
    """Download an object from S3 to a local file.

    :param key: S3 object key to download.
    :param dest_path: Local file path where the object will be saved.
    """
    cfg = get_config()
    s3 = get_s3_client()

    dest_path.parent.mkdir(parents=True, exist_ok=True)

    try:
        with dest_path.open("wb") as f:
            s3.download_fileobj(Bucket=cfg.bucket_name, Key=key, Fileobj=f)
    except (BotoCoreError, ClientError) as e:
        raise RuntimeError(f"Error downloading S3 object: {e}") from e


def delete_file(key: str) -> None:
    """Delete an object from S3.

    :param key: S3 object key to delete.
    """
    cfg = get_config()
    s3 = get_s3_client()

    try:
        s3.delete_object(Bucket=cfg.bucket_name, Key=key)
    except (BotoCoreError, ClientError) as e:
        raise RuntimeError(f"Error deleting S3 object: {e}") from e
