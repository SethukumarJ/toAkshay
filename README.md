# toAkshay

func (ra *SlackShotService) UploadToGCS(reportType, vendorId, geo string, screenshotFile []byte, bucketName string, htmlType string) (string, error) {
	objectName := fmt.Sprintf("report_alerts/%s/%s/%s_%s_%s.png", vendorId, htmlType, reportType, geo, time.Now().Format("2006-01-02"))

	// Set the retention policy for the object to automatically delete in 5 hours.
	object := ra.storageClient.Bucket(bucketName).Object(objectName)

	ctx := context.Background()
	wc := object.NewWriter(ctx)
	if _, err := io.Copy(wc, bytes.NewReader(screenshotFile)); err != nil {
		ra.logger.New(logger.LogEntry{
			Severity: logger.Error,
			Payload: map[string]interface{}{
				"error": err,
				"meta":  "Error while uploading screenshot",
			},
		})
		return "", err
	}
	if err := wc.Close(); err != nil {
		return "", err
	}

	// Create an ObjectAttrsToUpdate with a retention policy.
	attrsToUpdate := storage.ObjectAttrsToUpdate{
		ContentType: "image/png",
		CustomTime:  time.Now().Add((24 * time.Hour) * 15),
	}

	// Update the object's attributes to set the retention policy.
	if _, err := object.Update(ctx, attrsToUpdate); err != nil {
		ra.logger.New(logger.LogEntry{
			Severity: logger.Error,
			Payload: map[string]interface{}{
				"error": err,
				"meta":  "Error while updating object attributes while uploading screenshot",
			},
		})
		return "", err
	}

	url, err := ra.getSignedURL(bucketName, objectName, time.Duration((24*time.Hour)*15))
	if err != nil {
		ra.logger.New(logger.LogEntry{
			Severity: logger.Error,
			Payload: map[string]interface{}{
				"error": err,
				"meta":  "Error while getting signed URL",
			},
		})
		return "", err
	}

	return url, nil
}

func (client *SlackShotService) getSignedURL(bucket string, fileName string, expiration time.Duration) (url string, err error) {

	url, err = storage.SignedURL(bucket, fileName, &storage.SignedURLOptions{
		Method:         http.MethodGet,
		GoogleAccessID: "207791529245-compute@developer.gserviceaccount.com",
		Expires:        time.Now().Add(expiration),
		PrivateKey:     []byte("-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDR5+6EXyK52Way\nKm4dpDWY0l7I6Rnp3od6H6Npz1Yrgpq+bscXZLUCUBfCYA0inOEj6Xu9tk4+uuVr\nBAsj8wnmcD5UAPQO5+gzIm5P6yU+kY+skvHVkD6IU8a/ldQjwLQCXeZlahaN/Uqc\nnXYhfMvnFmVGsS7ZLoG1LQcH1DLQf/13u4X00kpKG5sYmnqMIsby4A08Ckpmaz4O\ncn/tUYCZfqEvLYkGWwk3ImfulBkr2egALbZ7lcwxAtkG3VFTcmbXbrPVO3ZFIWNq\nXhC2/eLIx5zrEtdHWrnMG78gZIIZqwAEtcaqMkCR/omDI3vLgWY8v8/nlhnyiOLI\nE9mVIX25AgMBAAECggEAHRtGgBqFFopRQEFmoDubUa6nhWqKtQWuJD7A39+So66U\n8J/XLE+YU1LHM0/dt7/97tyQfmpfh+P4rhHoeDqvqRbwEvadi8ztQ4dHDqlHsoO9\nYtZu1wi3yHBS2KbZEirFi1J5Vp9oXetn7jcIy3SiUvqls9txPfJspV0yDWGHuPuA\nyn0XDxvJZHcMROvFNQQrmshBm5uEPKHxnTtDJ/sI2smpVPiSlBVWz/rzfu6i0hlS\nOo+8UZxeWKFYSOypr9x3nBB29YT6BGf7ufS9h2zjMw8xjCzwzuZjRJHblmSi2Jg7\nkeo4MbkAAthuCBfncNncmnb3Ro0vQhx5R9TidWprtQKBgQD0Ks7bcz48ATw2+KJs\n/LlIe0+gfhFWOp6QlGwq7MmxYHrKxPH9+oJ+XIob/1LfpCAOwZbkEnKqGJcCOoWU\nZkN8tKN4RQEqmCIRNAHSLvwNhD8XdTaBSrrdSLh+Tj5CGXyarArbm2aSNBFN87r5\nPzAClrZb/lhXCHsScsZZ6iE+2wKBgQDcFBI/jFzIDLUwwyqTuKuqr5uRshTKSZRe\nP4dGMlIEydKaMqCq41vC7AxYA3CJ4O1X909/rMXfbSMk2Y5S3ssdFwauIaSwKpRT\n23h5y+A3ztseGn3RAojesWLHfqzeMBPWFo74bklfvUBICybUnUb0GtLDZHQWhibY\nj0Wj9hin+wKBgA5t7TWY1Oe05vsUrHymXsjCyMziRmIDKtW+f7n1rmG2IuuSwf5R\nbJ7NFzhaWWpwB5j3pdQqpu4Yb+woyzYe6QQYpMR5x3zd6r17hlQGhMzDsPrQ6Xyw\njuR+5LBKLXG4kd2OJ0IdJ+2h+BfUPIt4SX0NrQ84s73I+YT4lXJA3OAbAoGBALdh\nnicHvZQQSracGZFH0vuCIn5PxlUc5J14EC8k5QUKawuD3i8nDiIo8Mwx6YdqPjsL\nX1oCzEq1NRCSm65f6R2PP0i/zevhPwF1IjlS8b1vB1RZPLd5hjUR2D5lRoRJyW2e\nFHnb5BX7q2GcsTl+6E2lQDQCM11FYX8YOy45dSgbAoGAfxuh9n15K1Js+swle1HC\n7DdswZ7ckN22uz3xuOTnEMcO3myXh7zpDTq1D09mqCQPdFhsrGxZKDPfw/NFM9mA\n1Fza8LwziiUmxy6O2cNUSTxc3ImdgiGh96uzFH8BphhVZTOQkokYIaI66fNG0v1+\nTL08BmpWMxEwIJzCpShws0w=\n-----END PRIVATE KEY-----\n"),
	})
	return
}