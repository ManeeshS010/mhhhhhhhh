import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.math.BigInteger;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Date;
import java.util.List;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.jms.JmsException;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.test.util.ReflectionTestUtils;

import com.suntec.custom.dto.CtsMLNotificationMessage;
import com.suntec.custom.model.SGAlertView;
import com.suntec.custom.model.SGDeliveryMethod;
import com.suntec.custom.model.SGEmailAlert;
import com.suntec.custom.repository.FollowUpLogRepository;
import com.suntec.custom.repository.SGDeliveryMethodRepository;
import com.suntec.custom.repository.SGEmailAlertRepository;

@ExtendWith(MockitoExtension.class)
class SGAlertProcessorTest {

    @Mock
    private JmsTemplate jmsTemplate;

    @Mock
    private SGUtils sGUtils;

    @Mock
    private FollowUpLogRepository followUpLogRepository;

    @Mock
    private SGDeliveryMethodRepository sGDeliveryMethodRepository;

    @Mock
    private SGEmailAlertRepository sGEmailAlertRepository;

    @InjectMocks
    private SGAlertProcessor sgAlertProcessor;

    private SGAlertView sgAlertView;

    @BeforeEach
    void setUp() {
        // Set @Value properties using ReflectionTestUtils
        ReflectionTestUtils.setField(sgAlertProcessor, "cobaltRequestQueue", "test.queue");
        ReflectionTestUtils.setField(sgAlertProcessor, "CRET", "CRET");
        ReflectionTestUtils.setField(sgAlertProcessor, "nickelIRT", "IRT");
        ReflectionTestUtils.setField(sgAlertProcessor, "codDemande", "COD");
        ReflectionTestUtils.setField(sgAlertProcessor, "canalEntree", "CANAL");
        ReflectionTestUtils.setField(sgAlertProcessor, "sendingBIC", "BICE");
        ReflectionTestUtils.setField(sgAlertProcessor, "lib20Trn", ":20:");
        ReflectionTestUtils.setField(sgAlertProcessor, "lib21Reltrn", ":21:");
        ReflectionTestUtils.setField(sgAlertProcessor, "txt79msg199_1", ":79:");
        ReflectionTestUtils.setField(sgAlertProcessor, "txt79msg199_n", ":79:");

        // Initialize test data
        sgAlertView = new SGAlertView();
        sgAlertView.setSavId(12345L);
        sgAlertView.setSavBillNo("INV001");
        sgAlertView.setSavBillDate("2024-01-15");
        sgAlertView.setSavBicCode("TESTBIC");
        sgAlertView.setSavBranchCode("BR001");
        sgAlertView.setSavFollowupYn("Y");
        sgAlertView.setSavMessage("Test message <InvoiceNo> <InvoiceDate> &nbsp; <b>HTML</b>");
    }

    @Test
    void testProcess_FollowUpNotRequired() throws Exception {
        sgAlertView.setSavFollowupYn("N");
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("4"), eq(12345L), eq("Follow up not required"), any(Date.class));
    }

    @Test
    void testProcess_SwiftMode() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));

        sgAlertProcessor.process(sgAlertView);

        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(followUpLogRepository).updateFollowUpStatus(
            eq("2"), eq(12345L), isNull(), any(Date.class));
    }

    @Test
    void testProcess_EmailMode() throws Exception {
        sgAlertView.setSavAlertMode("E");
        setupEmailMocks(HttpStatus.ACCEPTED);

        sgAlertProcessor.process(sgAlertView);

        verify(sGDeliveryMethodRepository).findDelMethdByBillNo("INV001");
        verify(followUpLogRepository).updateFollowUpStatus(
            eq("2"), eq(12345L), isNull(), any(Date.class));
    }

    @Test
    void testProcess_BothMode() throws Exception {
        sgAlertView.setSavAlertMode("B");
        setupEmailMocks(HttpStatus.ACCEPTED);

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));

        sgAlertProcessor.process(sgAlertView);

        verify(jmsTemplate).convertAndSend(anyString(), anyString());
        verify(sGDeliveryMethodRepository).findDelMethdByBillNo("INV001");
    }

    @Test
    void testProcess_InvalidAlertMode() throws Exception {
        sgAlertView.setSavAlertMode("X");

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("4"), eq(12345L), eq("Invalid Alert Mode"), any(Date.class));
    }

    @Test
    void testProcessSwift_EmptyBicCode() throws Exception {
        sgAlertView.setSavAlertMode("S");
        sgAlertView.setSavBicCode("");

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("Destination BIC is empty"), any(Date.class));
        verify(jmsTemplate, never()).convertAndSend(anyString(), anyString());
    }

    @Test
    void testProcessSwift_NullBicCode() throws Exception {
        sgAlertView.setSavAlertMode("S");
        sgAlertView.setSavBicCode(null);

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("Destination BIC is empty"), any(Date.class));
        verify(jmsTemplate, never()).convertAndSend(anyString(), anyString());
    }

    @Test
    void testProcessSwift_JmsException() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));
        doThrow(new JmsException("JMS Error occurred while sending message to queue") {})
            .when(jmsTemplate).convertAndSend(anyString(), anyString());

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("JMS Error occurred while sending message to queue"), any(Date.class));
    }

    @Test
    void testProcessSwift_JmsExceptionLongMessage() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));
        
        String longError = "JMS Error: " + "x".repeat(100);
        doThrow(new JmsException(longError) {})
            .when(jmsTemplate).convertAndSend(anyString(), anyString());

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq(longError.substring(0, 90)), any(Date.class));
    }

    @Test
    void testProcessSwift_MessageReplacement() throws Exception {
        sgAlertView.setSavAlertMode("S");
        sgAlertView.setSavMessage("Invoice: <InvoiceNo>\nDate: <InvoiceDate>\r\n&nbsp;<tag>content</tag>  double  space");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils).splitNoWords(
            eq("Invoice: INV001 Date: 2024-01-15 content double space"), eq(50));
    }

    @Test
    void testProcessEmail_DeliveryTypeW() throws Exception {
        sgAlertView.setSavAlertMode("E");
        setupEmailMocks(HttpStatus.ACCEPTED);

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils).sendCtsml(any(CtsMLNotificationMessage.class), eq("INV001"), eq("INV"), eq("W"));
        verify(followUpLogRepository).updateFollowUpStatus(
            eq("2"), eq(12345L), isNull(), any(Date.class));
    }

    @Test
    void testProcessEmail_DeliveryTypeB() throws Exception {
        sgAlertView.setSavAlertMode("E");
        setupEmailMocksWithDeliveryType("B", HttpStatus.ACCEPTED);

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils).sendCtsml(any(CtsMLNotificationMessage.class), eq("INV001"), eq("INV"), eq("W"));
        verify(followUpLogRepository).updateFollowUpStatus(
            eq("2"), eq(12345L), isNull(), any(Date.class));
    }

    @Test
    void testProcessEmail_DeliveryTypeNotWOrB() throws Exception {
        sgAlertView.setSavAlertMode("E");
        setupEmailMocksWithDeliveryType("X", HttpStatus.ACCEPTED);

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils, never()).sendCtsml(any(), anyString(), anyString(), anyString());
        verify(followUpLogRepository, never()).updateFollowUpStatus(
            eq("2"), any(), any(), any(Date.class));
    }

    @Test
    void testProcessEmail_FailedResponse() throws Exception {
        sgAlertView.setSavAlertMode("E");
        setupEmailMocks(HttpStatus.BAD_REQUEST);

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("Error response body"), any(Date.class));
    }

    @Test
    void testProcessEmail_FailedResponseLongBody() throws Exception {
        sgAlertView.setSavAlertMode("E");
        
        String longBody = "Error: " + "x".repeat(100);
        setupEmailMocksWithResponseBody(HttpStatus.BAD_REQUEST, longBody);

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq(longBody.substring(0, 90)), any(Date.class));
    }

    @Test
    void testProcessEmail_NoDeliveryMethods() throws Exception {
        sgAlertView.setSavAlertMode("E");

        SGEmailAlert emailAlert = new SGEmailAlert();
        emailAlert.setMailBody("Body");
        emailAlert.setMailSubject("Subject");

        when(sGEmailAlertRepository.getOne(any(BigInteger.class))).thenReturn(emailAlert);
        when(sGDeliveryMethodRepository.findDelMethdByBillNo("INV001"))
            .thenReturn(new ArrayList<>());

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("No Delivery Method configured"), any(Date.class));
    }

    @Test
    void testProcessEmail_NullDeliveryMethods() throws Exception {
        sgAlertView.setSavAlertMode("E");

        SGEmailAlert emailAlert = new SGEmailAlert();
        emailAlert.setMailBody("Body");
        emailAlert.setMailSubject("Subject");

        when(sGEmailAlertRepository.getOne(any(BigInteger.class))).thenReturn(emailAlert);
        when(sGDeliveryMethodRepository.findDelMethdByBillNo("INV001")).thenReturn(null);

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("No Delivery Method configured"), any(Date.class));
    }

    @Test
    void testProcessEmail_NullEmailAlert() throws Exception {
        sgAlertView.setSavAlertMode("E");

        when(sGEmailAlertRepository.getOne(any(BigInteger.class))).thenReturn(null);

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("Email Alert missing"), any(Date.class));
    }

    @Test
    void testProcessEmail_ExceptionThrown() throws Exception {
        sgAlertView.setSavAlertMode("E");

        when(sGEmailAlertRepository.getOne(any(BigInteger.class)))
            .thenThrow(new RuntimeException("Database error"));

        sgAlertProcessor.process(sgAlertView);

        verify(followUpLogRepository).updateFollowUpStatus(
            eq("3"), eq(12345L), eq("Email Alert missing"), any(Date.class));
    }

    @Test
    void testGenerate199Message_MultipleSplits() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader("199", "12345", "TESTBIC", "BR001"))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), eq(50)))
            .thenReturn(Arrays.asList("split1", "split2", "split3"));

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils, atLeastOnce()).padRight(eq(":79:"), eq(30));
        verify(sGUtils, atLeastOnce()).padRight(eq(":79:2"), eq(30));
        verify(sGUtils, atLeastOnce()).padRight(eq(":79:3"), eq(30));
    }

    @Test
    void testGenerate199Message_EmptySplits() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader("199", "12345", "TESTBIC", "BR001"))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), eq(50))).thenReturn(new ArrayList<>());

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils).padRight(contains("003"), eq(3));
    }

    @Test
    void testGenerate199Message_NullSplits() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader("199", "12345", "TESTBIC", "BR001"))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), eq(50))).thenReturn(null);

        sgAlertProcessor.process(sgAlertView);

        verify(sGUtils).padRight(contains("003"), eq(3));
    }

    @Test
    void testProcess_ReturnsNull() throws Exception {
        sgAlertView.setSavAlertMode("S");

        when(sGUtils.generateHeader(anyString(), anyString(), anyString(), anyString()))
            .thenReturn("HEADER");
        when(sGUtils.padRight(anyString(), anyInt())).thenReturn("PADDED");
        when(sGUtils.splitNoWords(anyString(), anyInt())).thenReturn(Arrays.asList("split1"));

        SGAlertView result = sgAlertProcessor.process(sgAlertView);

        assertNull(result);
    }

    // Helper methods
    private void setupEmailMocks(HttpStatus status) throws Exception {
        setupEmailMocksWithDeliveryType("W", status);
    }

    private void setupEmailMocksWithDeliveryType(String deliveryType, HttpStatus status) throws Exception {
        setupEmailMocksWithResponseBody(status, "Error response body", deliveryType);
    }

    private void setupEmailMocksWithResponseBody(HttpStatus status, String body) throws Exception {
        setupEmailMocksWithResponseBody(status, body, "W");
    }

    private void setupEmailMocksWithResponseBody(HttpStatus status, String body, String deliveryType) throws Exception {
        SGEmailAlert emailAlert = new SGEmailAlert();
        emailAlert.setMailBody("Test Body");
        emailAlert.setMailSubject("Test Subject");

        SGDeliveryMethod deliveryMethod = new SGDeliveryMethod();
        deliveryMethod.setDeliveryType(deliveryType);

        List<SGDeliveryMethod> deliveryMethods = Arrays.asList(deliveryMethod);

        CtsMLNotificationMessage ctsml = new CtsMLNotificationMessage();
        
        ResponseEntity<String> response = new ResponseEntity<>(body, status);

        when(sGEmailAlertRepository.getOne(any(BigInteger.class))).thenReturn(emailAlert);
        when(sGDeliveryMethodRepository.findDelMethdByBillNo("INV001")).thenReturn(deliveryMethods);
        when(sGUtils.prepareCtsml(any(), any(), anyString(), anyString(), isNull(), isNull(), isNull(), eq("W")))
            .thenReturn(Arrays.asList(ctsml));
        when(sGUtils.sendCtsml(any(CtsMLNotificationMessage.class), anyString(), anyString(), anyString()))
            .thenReturn(response);
    }
}
