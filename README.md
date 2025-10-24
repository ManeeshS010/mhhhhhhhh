package com.suntec.custom.service;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.sql.Timestamp;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import com.suntec.custom.config.MyBusinessException;
import com.suntec.custom.dto.CtsMLNotificationMessage;
import com.suntec.custom.model.SGPwdResetAlertView;
import com.suntec.custom.repository.SGPwdResetAlertRepository;

@ExtendWith(MockitoExtension.class)
class SGPwdResetAlertProcessorTest {

    @Mock
    private SGUtils sGUtils;

    @Mock
    private SGPwdResetAlertRepository pwdResetAlertRepository;

    @InjectMocks
    private SGPwdResetAlertProcessor processor;

    private SGPwdResetAlertView testView;
    private CtsMLNotificationMessage testCtsml;

    @BeforeEach
    void setUp() {
        testView = new SGPwdResetAlertView();
        testView.setSprav_cust_id("CUST123");
        
        testCtsml = new CtsMLNotificationMessage();
    }

    @Test
    void testProcess_Success() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Success", HttpStatus.ACCEPTED));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(sGUtils).preparePwdResetMail(testView);
        verify(sGUtils).sendCtsml(testCtsml, "CUST123", "INV", "W");
        verify(pwdResetAlertRepository).updatePwdResetDate(any(Timestamp.class), eq("CUST123"));
    }

    @Test
    void testProcess_WithNullView() throws Exception {
        // Act
        SGPwdResetAlertView result = processor.process(null);

        // Assert
        assertNull(result);
        verify(sGUtils, never()).preparePwdResetMail(any());
        verify(sGUtils, never()).sendCtsml(any(), any(), any(), any());
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcessEmail_Success_StatusCode202() {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Success", HttpStatus.ACCEPTED));

        // Act
        assertDoesNotThrow(() -> processor.process(testView));

        // Assert
        verify(pwdResetAlertRepository).updatePwdResetDate(any(Timestamp.class), eq("CUST123"));
    }

    @Test
    void testProcessEmail_Failure_StatusCodeNot202() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Error message", HttpStatus.BAD_REQUEST));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcessEmail_ExceptionDuringSendCtsml() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenThrow(new RuntimeException("Network error"));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcessEmail_ExceptionDuringPrepareMail() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView))
            .thenThrow(new RuntimeException("Preparation error"));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(sGUtils, never()).sendCtsml(any(), any(), any(), any());
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcessEmail_StatusCode200_NotAccepted() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("OK", HttpStatus.OK));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcessEmail_StatusCode500_ServerError() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Server Error", HttpStatus.INTERNAL_SERVER_ERROR));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testGetTimeStamp() {
        // Act
        Timestamp ts1 = processor.getTimeStamp();
        
        // Small delay to ensure different timestamps
        try {
            Thread.sleep(2);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        Timestamp ts2 = processor.getTimeStamp();

        // Assert
        assertNotNull(ts1);
        assertNotNull(ts2);
        assertTrue(ts2.getTime() >= ts1.getTime());
    }

    @Test
    void testProcessEmail_NullPointerExceptionDuringUpdate() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Success", HttpStatus.ACCEPTED));
        doThrow(new NullPointerException("Database error"))
            .when(pwdResetAlertRepository).updatePwdResetDate(any(Timestamp.class), eq("CUST123"));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository).updatePwdResetDate(any(Timestamp.class), eq("CUST123"));
    }

    @Test
    void testProcessEmail_MultipleCustomerIds() throws Exception {
        // Test with different customer ID
        testView.setSprav_cust_id("CUST456");
        
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST456", "INV", "W"))
            .thenReturn(new ResponseEntity<>("Success", HttpStatus.ACCEPTED));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository).updatePwdResetDate(any(Timestamp.class), eq("CUST456"));
    }

    @Test
    void testProcessEmail_ResponseEntityWithNullBody() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView)).thenReturn(testCtsml);
        when(sGUtils.sendCtsml(testCtsml, "CUST123", "INV", "W"))
            .thenReturn(new ResponseEntity<>(null, HttpStatus.BAD_REQUEST));

        // Act
        SGPwdResetAlertView result = processor.process(testView);

        // Assert
        assertNull(result);
        verify(pwdResetAlertRepository, never()).updatePwdResetDate(any(), any());
    }

    @Test
    void testProcess_CatchesAllExceptions() throws Exception {
        // Arrange
        when(sGUtils.preparePwdResetMail(testView))
            .thenThrow(new OutOfMemoryError("Memory error"));

        // Act & Assert - Should catch the error and return null
        SGPwdResetAlertView result = processor.process(testView);
        assertNull(result);
    }
}
