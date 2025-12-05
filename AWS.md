# AWS Code
This markdown file contains code relating to the use of AWS services.


``` python
import boto3
from botocore.config import Config
from botocore.exceptions import ClientError

class BucketABC(ABC):
    """Bucket abstract base class."""

    @abstractmethod
    def put(self, key: str, data: bytes) -> None:
        """Put data into bucket."""
    @abstractmethod
    def get(self, key: str) -> bytes:
        """Get data from bucket."""
    @abstractmethod
    def delete(self, key: str) -> None:
        """Delete data from bucket."""
    @abstractmethod
    def exists(self, key: str) -> bool:
        """Check if bucket exists."""
    @abstractmethod
    def verify(self, create_if_missing: bool = False) -> bool:
        """Verify bucket connection."""

class S3Bucket(BucketABC):
    """S3 Bucket Implementation."""

    def __init__(  # noqa: PLR0913
        self,
        *,
        bucket: str,
        prefix: str = "",
        region_name: str = "eu-west-1",
        endpoint_url: str | None = None,   # set for LocalStack
        aws_access_key_id: str | None = None,
        aws_secret_access_key: str | None = None,
        path_style: bool = True,           # safer with emulators
    ) -> None:
        """Initialise S3 Bucket client."""
        self.bucket = bucket
        self.prefix = prefix
        self.region_name = region_name
        self.endpoint_url = endpoint_url

        cfg = Config(s3={"addressing_style": "path"} if path_style else {})
        self.s3 = boto3.client(
            "s3",
            region_name=region_name,
            endpoint_url=endpoint_url,
            aws_access_key_id=aws_access_key_id,
            aws_secret_access_key=aws_secret_access_key,
            config=cfg,
        )
        logger.debug(
            "S3Bucket client initialised (bucket='{}', region='{}', endpoint='{}', "
            "prefix='{}')",
            self.bucket, self.region_name, self.endpoint_url, self.prefix or "",
        )

    def _k(self, key: str) -> str:
        key = _norm(key)
        return f"{self.prefix}{key}" if self.prefix else key

    def put(self, key: str, data: bytes) -> None:
        """Put data into bucket."""
        k = self._k(key)
        self.s3.put_object(Bucket=self.bucket, Key=k, Body=data)
        logger.debug("PUT s3://{}/{} ({} bytes)", self.bucket, k, len(data))

    def get(self, key: str) -> bytes:
        """Get data from bucket."""
        k = self._k(key)
        obj = self.s3.get_object(Bucket=self.bucket, Key=k)
        data = obj["Body"].read()
        logger.debug("GET s3://{}/{} ({} bytes)", self.bucket, k, len(data))
        return data

    def delete(self, key: str) -> None:
        """Delete data from bucket."""
        k = self._k(key)
        self.s3.delete_object(Bucket=self.bucket, Key=k)
        logger.debug("DELETE s3://{}/{}", self.bucket, k)

    def exists(self, key: str) -> bool:
        """Check if bucket exists."""
        k = self._k(key)
        try:
            self.s3.head_object(Bucket=self.bucket, Key=k)
            logger.trace("EXISTS s3://{}/{} -> True", self.bucket, k)
            return True
        except ClientError:
            logger.trace("EXISTS s3://{}/{} -> False", self.bucket, k)
            return False

    def verify(self, create_if_missing: bool = False) -> bool:
        """Check access to the bucket (and optionally create it)."""
        try:
            self.s3.head_bucket(Bucket=self.bucket)
            # tiny list to catch perms against the (optional) prefix
            self.s3.list_objects_v2(Bucket=self.bucket, MaxKeys=1,
                                    Prefix=self.prefix or "")
            logger.info(
                "✅ S3 connection verified (bucket='{}', region='{}', endpoint='{}')",
                self.bucket, self.region_name, self.endpoint_url,
            )
            return True
        except ClientError as e:
            code = e.response.get("Error", {}).get("Code", "")
            if create_if_missing and code in ("404", "NoSuchBucket", "NotFound"):
                self.s3.create_bucket(Bucket=self.bucket)
                logger.info(
                    "🪣 Created missing bucket '{}' (region='{}', endpoint='{}')",
                    self.bucket, self.region_name, self.endpoint_url,
                )
                return True
            logger.exception(
                "S3 verify failed for bucket='{}' (code={})", self.bucket, code
            )
            raise
```