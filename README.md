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
		GoogleAccessID: os.Getenv(),
		Expires:        time.Now().Add(expiration),
		PrivateKey:     []byte(os.Getenv()),
	})
	return
}