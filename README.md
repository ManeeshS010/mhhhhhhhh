import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

import java.io.File;
import java.math.BigInteger;
import java.util.Date;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.io.TempDir;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.web.client.RestTemplate;

import com.suntec.custom.dto.CtsMLAcknowledgementMessage;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.CorrespondenceStatusComment;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet.DocumentInfo;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet.DocumentInfo.AddresseeMediaSet;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet.DocumentInfo.AddresseeMediaSet.MediaInfo;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet.DocumentInfo.DocumentClassificationInfo;
import com.suntec.custom.dto.CtsMLAcknowledgementMessage.DocumentStatusInfo.DocumentSet.DocumentInfo.DocumentClassificationInfo.DocumentCode;
import com.suntec.custom.repository.BillReportRepository;
import com.suntec.custom.repository.FollowUpLogRepository;

@ExtendWith(MockitoExtension.class)
class SGStatusProcessorTest {

    @Mock
    private SGUtils sGUtils;

    @Mock
    private BillReportRepository billReportRepository;

    @Mock
    private JmsTemplate jmsTemplate;

    @Mock
    private FollowUpLogRepository followUpLogRepository;

    @Mock
    private RestTemplate restTemplate;

    @Mock
    private StepContribution stepContribution;

    @Mock
    private ChunkContext chunkContext;

    @InjectMocks
    private SGStatusProcessor processor;

    @TempDir
    File tempDir;

    @BeforeEach
    void setUp() {
        ReflectionTestUtils.setField(processor, "cobaltResponseQueue", "test.queue");
        ReflectionTestUtils.setField(processor, "sirocco_scope", "test_scope");
        ReflectionTestUtils.setField(processor, "sirocco_notification_path", "http://test.com/notify");
        ReflectionTestUtils.setField(processor, "sirocco_mediaName_EMAIL", "EMAIL");
        ReflectionTestUtils.setField(processor, "sirocco_Datapath", tempDir.getAbsolutePath());
    }

    @Test
    void testExecute_NoSiroccoMessages_NoCobaltMessages() throws Exception {
        // Given
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(sGUtils, times(1)).getToken(anyString());
        verify(jmsTemplate, times(1)).receiveAndConvert(anyString());
    }

    @Test
    void testExecute_WithEmailMediaType_Success() throws Exception {
        // Given
        String xmlResponse = createValidXmlResponse("EMAIL", "123456789", "SENT", "Success");
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespEmail(
                eq("SENT"), 
                eq(BigInteger.valueOf(123456789L)), 
                eq("Success"));
    }

    @Test
    void testExecute_WithPostMediaType_Success() throws Exception {
        // Given
        String xmlResponse = createValidXmlResponse("POST", "987654321", "DELIVERED", "Delivered successfully");
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespPost(
                eq("DELIVERED"), 
                eq(BigInteger.valueOf(987654321L)), 
                eq("Delivered successfully"));
    }

    @Test
    void testExecute_WithLongErrorMessage_Truncated() throws Exception {
        // Given
        String longError = "A".repeat(400);
        String xmlResponse = createValidXmlResponse("EMAIL", "111111111", "ERROR", longError);
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespEmail(
                eq("ERROR"), 
                eq(BigInteger.valueOf(111111111L)), 
                eq("A".repeat(300)));
    }

    @Test
    void testExecute_WithNullCommentInfo_Success() throws Exception {
        // Given
        String xmlResponse = createValidXmlResponseWithNullComment("EMAIL", "222222222", "PENDING");
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespEmail(
                eq("PENDING"), 
                eq(BigInteger.valueOf(222222222L)), 
                isNull());
    }

    @Test
    void testExecute_MediaNameException_UsesDefaultEmail() throws Exception {
        // Given
        String xmlResponse = createValidXmlResponseWithoutMedia("333333333", "SENT", "Success");
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespEmail(
                eq("SENT"), 
                eq(BigInteger.valueOf(333333333L)), 
                eq("Success"));
    }

    @Test
    void testExecute_SiroccoException_RetriesUpTo5Times() throws Exception {
        // Given
        when(sGUtils.getToken(anyString())).thenThrow(new RuntimeException("Connection error"));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(sGUtils, times(5)).getToken(anyString());
    }

    @Test
    void testExecute_WithCobaltResponse_ValidLength() throws Exception {
        // Given
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        
        String cobaltResponse = createCobaltResponse(1234567890, "Error message");
        when(jmsTemplate.receiveAndConvert(anyString()))
                .thenReturn(cobaltResponse)
                .thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(followUpLogRepository, times(1)).updateFollowUpStatus(
                eq("2"), 
                eq(1234567890), 
                anyString(), 
                any(Date.class));
    }

    @Test
    void testExecute_WithCobaltResponse_ShortLength() throws Exception {
        // Given
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        
        String shortResponse = "Short message";
        when(jmsTemplate.receiveAndConvert(anyString()))
                .thenReturn(shortResponse)
                .thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(followUpLogRepository, never()).updateFollowUpStatus(
                anyString(), anyInt(), anyString(), any(Date.class));
    }

    @Test
    void testExecute_CobaltException_RetriesUpTo5Times() throws Exception {
        // Given
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString()))
                .thenThrow(new RuntimeException("JMS error"));

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(jmsTemplate, times(5)).receiveAndConvert(anyString());
    }

    @Test
    void testExecute_InvalidXmlResponse_ThrowsException() throws Exception {
        // Given
        String invalidXml = "Invalid XML content";
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(invalidXml.getBytes(), HttpStatus.OK));
        when(jmsTemplate.receiveAndConvert(anyString())).thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(sGUtils, times(5)).getToken(anyString());
    }

    @Test
    void testExecute_MultipleMessages_ProcessesAll() throws Exception {
        // Given
        String xmlResponse1 = createValidXmlResponse("EMAIL", "111111111", "SENT", "First");
        String xmlResponse2 = createValidXmlResponse("POST", "222222222", "DELIVERED", "Second");
        
        when(sGUtils.getToken(anyString())).thenReturn("test_token");
        when(sGUtils.getRestTemplate()).thenReturn(restTemplate);
        when(restTemplate.exchange(anyString(), any(HttpMethod.class), any(HttpEntity.class), 
                eq(byte[].class), anyString()))
                .thenReturn(new ResponseEntity<>(xmlResponse1.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(xmlResponse2.getBytes(), HttpStatus.OK))
                .thenReturn(new ResponseEntity<>(null, HttpStatus.OK));
        
        String cobaltResponse1 = createCobaltResponse(1111111111, "Cobalt1");
        String cobaltResponse2 = createCobaltResponse(2222222222L, "Cobalt2");
        when(jmsTemplate.receiveAndConvert(anyString()))
                .thenReturn(cobaltResponse1)
                .thenReturn(cobaltResponse2)
                .thenReturn(null);

        // When
        RepeatStatus result = processor.execute(stepContribution, chunkContext);

        // Then
        assertNull(result);
        verify(billReportRepository, times(1)).updateCommRespEmail(anyString(), any(), anyString());
        verify(billReportRepository, times(1)).updateCommRespPost(anyString(), any(), anyString());
        verify(followUpLogRepository, times(2)).updateFollowUpStatus(anyString(), anyInt(), anyString(), any(Date.class));
    }

    // Helper methods to create test data
    private String createValidXmlResponse(String mediaType, String docCode, String status, String comment) {
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<CtsMLAcknowledgementMessage>\n" +
                "  <DocumentStatusInfo>\n" +
                "    <CorrespondenceDocumentStatus>" + status + "</CorrespondenceDocumentStatus>\n" +
                "    <CorrespondenceStatusComment>\n" +
                "      <CommentInfo>" + comment + "</CommentInfo>\n" +
                "    </CorrespondenceStatusComment>\n" +
                "    <DocumentSet>\n" +
                "      <DocumentInfo>\n" +
                "        <DocumentClassificationInfo>\n" +
                "          <DocumentCode>\n" +
                "            <Code>" + docCode + "</Code>\n" +
                "          </DocumentCode>\n" +
                "        </DocumentClassificationInfo>\n" +
                "        <AddresseeMediaSet>\n" +
                "          <MediaInfo>\n" +
                "            <MediaName>" + mediaType + "</MediaName>\n" +
                "          </MediaInfo>\n" +
                "        </AddresseeMediaSet>\n" +
                "      </DocumentInfo>\n" +
                "    </DocumentSet>\n" +
                "  </DocumentStatusInfo>\n" +
                "</CtsMLAcknowledgementMessage>";
    }

    private String createValidXmlResponseWithNullComment(String mediaType, String docCode, String status) {
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<CtsMLAcknowledgementMessage>\n" +
                "  <DocumentStatusInfo>\n" +
                "    <CorrespondenceDocumentStatus>" + status + "</CorrespondenceDocumentStatus>\n" +
                "    <DocumentSet>\n" +
                "      <DocumentInfo>\n" +
                "        <DocumentClassificationInfo>\n" +
                "          <DocumentCode>\n" +
                "            <Code>" + docCode + "</Code>\n" +
                "          </DocumentCode>\n" +
                "        </DocumentClassificationInfo>\n" +
                "        <AddresseeMediaSet>\n" +
                "          <MediaInfo>\n" +
                "            <MediaName>" + mediaType + "</MediaName>\n" +
                "          </MediaInfo>\n" +
                "        </AddresseeMediaSet>\n" +
                "      </DocumentInfo>\n" +
                "    </DocumentSet>\n" +
                "  </DocumentStatusInfo>\n" +
                "</CtsMLAcknowledgementMessage>";
    }

    private String createValidXmlResponseWithoutMedia(String docCode, String status, String comment) {
        return "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<CtsMLAcknowledgementMessage>\n" +
                "  <DocumentStatusInfo>\n" +
                "    <CorrespondenceDocumentStatus>" + status + "</CorrespondenceDocumentStatus>\n" +
                "    <CorrespondenceStatusComment>\n" +
                "      <CommentInfo>" + comment + "</CommentInfo>\n" +
                "    </CorrespondenceStatusComment>\n" +
                "    <DocumentSet>\n" +
                "      <DocumentInfo>\n" +
                "        <DocumentClassificationInfo>\n" +
                "          <DocumentCode>\n" +
                "            <Code>" + docCode + "</Code>\n" +
                "          </DocumentCode>\n" +
                "        </DocumentClassificationInfo>\n" +
                "      </DocumentInfo>\n" +
                "    </DocumentSet>\n" +
                "  </DocumentStatusInfo>\n" +
                "</CtsMLAcknowledgementMessage>";
    }

    private String createCobaltResponse(long id, String message) {
        StringBuilder sb = new StringBuilder();
        sb.append("HEADER_CONTENT___"); // 18 chars
        sb.append(String.format("%10d", id)); // positions 19-28 (10 chars)
        sb.append("_".repeat(1134)); // padding to reach position 1163
        sb.append(message);
        sb.append("_".repeat(Math.max(0, 1251 - sb.length()))); // ensure length > 1250
        return sb.toString();
    }
}
